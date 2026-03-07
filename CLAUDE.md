# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

All builds require macOS 15+ on Apple Silicon. `xcrun clang` resolves the correct SDK automatically.

```bash
# Static pipeline (weights baked into MIL kernels, recompiles every ~10 steps)
cd training && make train_large
./train_large --model stories110M.bin --steps 100 --lr 1e-4
./train_large --resume   # resume from checkpoint

# Static + ANE extras (classifier/softmax/rmsnorm_bwd on ANE, 14% faster per-step)
make train_large_ane
./train_large_ane stories110M.bin 256 100 1e-4
./train_large_ane --no-ane-extras   # fall back to CPU for debugging

# Dynamic pipeline (compile 9 kernels once, no restarts — fastest wall time)
cd training/training_dynamic && make train
./train --scratch
./train --steps 200 --lr 1e-4

# Python bridge dylib
cd bridge && make

# Individual test probes
cd training && make test_rmsnorm_bwd && ./test_rmsnorm_bwd
cd training && make test_classifier  && ./test_classifier
cd training && make probes           # builds all 4 test_*.m probes

# Tokenize data
cd training && make tokenize         # runs tokenize.py

# Training data download
cd training && bash download_data.sh  # produces tinystories_data00.bin (~41MB)

# Dashboard (requires: pip install blessed psutil numpy)
sudo python3 training/dashboard.py           # static pipeline
sudo python3 training/dashboard.py --dynamic # dynamic pipeline

# Clean
make clean
```

## Architecture

### Three Training Pipelines

**1. Static Baseline (`training/train_large.m`)**
Weights baked as fp16 constants into MIL kernels. Triggers process restart (with checkpoint) every ~100 compiles to work around the ANE ~119 compile-per-process limit. 6 ANE kernels per layer x 12 layers + 1 classifier = 73 kernel types per batch. Header chain: `train_large.m` -> `stories_io.h` -> `stories_config.h`.

**2. Static + ANE Extras (`training/train_large_ane.m`)**
Extends the static baseline with additional ANE offload: classifier forward (32K conv), softmax, and RMSNorm backward. Defined in `ane_classifier.h` and `ane_rmsnorm_bwd.h`. Use `--no-ane-extras` to disable. Header chain adds `ane_rmsnorm_bwd.h` and `ane_classifier.h`.

**3. Dynamic Weight Pipeline (`training/training_dynamic/`)**
Weights pass through IOSurface spatial dimension alongside activations — activations at `sp[0:SEQ]`, weights packed at `sp[SEQ:]`. Compiles only 9 shared kernels once at startup, serving all 12 layers without recompilation. Self-contained: `train.m` -> `mil_dynamic.h` -> `io.h` -> `config.h` + `cpu_ops.h`.

### ANE Private API Layer (`training/ane_runtime.h`)

Used only by the **`train`** target (early prototype). Wraps the private `AppleNeuralEngine.framework` via `dlopen` + `objc_msgSend`:
- `_ANEInMemoryModelDescriptor` — compiles MIL text + weight blobs to ANE program
- `_ANEInMemoryModel` — loaded model, exposes `evaluateWithQoS:options:request:error:`
- `_ANERequest` / `_ANEIOSurfaceObject` — per-dispatch request with IOSurface bindings

The **static pipelines** (`train_large`, `train_large_ane`) replicate this pattern inline in `stories_io.h` using `compile_kern_mil_w()` and `ane_eval()`. `ane_eval()` returns `bool`; callers track `step_ok &= ane_eval(...)` and skip Adam updates on failure.

### Tensor Format

All ANE I/O uses IOSurface with layout `[1, channels, 1, spatial]` in fp16. CPU weights are fp32, converted on write via NEON vectorized `cvt_f32_f16()`. The channel-first layout eliminates transpose overhead vs. the standard `[N, H, W, C]` format.

### Weight Blobs (BLOBFILE format)

`build_blob()`, `build_blob_t()` (transposed), `build_blob_fp16()` in `stories_io.h` construct the proprietary BLOBFILE binary format: 128-byte header (`0xDEADBEEF` magic at offset 64, weight size at offset 72, data offset at offset 80) followed by fp16 weights. These are passed as `NSData` to `_ANEInMemoryModelDescriptor`.

### MIL Program Generation

MIL (Model Intermediate Language) programs are constructed as `NSString` at runtime using helpers in `stories_mil.h` (static) and `mil_dynamic.h` (dynamic). Linear layers become `conv` ops (ANE's native operation); attention uses `matmul`; element-wise ops use `mul`/`add`/`softmax`. The MIL version header (`program(1.3)`) and buildInfo dict must match the installed CoreML version.

### Temp Directory Naming

Each `compile_kern_mil_w()` call creates `ANE_<pid>_<seq>_<hash>` in `/tmp` for the MIL + weights. The `g_compile_seq` atomic counter ensures uniqueness across concurrent calls.

### Static Pipeline Header Dependencies

```
train_large.m / train_large_ane.m
  +-- stories_io.h        IOSurface helpers, blob builders, NEON conversion, kernel compile/eval
  |     +-- stories_config.h  Model constants (DIM=768, HEADS=12, SEQ=256, VOCAB=32000, NLAYERS=12),
  |                           structs (LayerWeights, LayerAdam, LayerActs, LayerGrads, LayerKernels),
  |                           xmf()/xcf() OOM-safe alloc helpers, ane_init() via dispatch_once
  +-- stories_mil.h       MIL generators for 6 kernel types (fwdAttn, fwdFFN, ffnBwd,
  |                       sdpaBwd1, sdpaBwd2, qkvBwd), mask blob
  +-- stories_cpu_ops.h   vDSP-vectorized RMSNorm fwd/bwd, cross-entropy, Adam optimizer,
  |                       embed_lookup/embed_backward with VOCAB bounds clamp
  [+ane only]
  +-- ane_rmsnorm_bwd.h   ANE RMSNorm backward kernel
  +-- ane_classifier.h    ANE classifier + softmax kernels
```

### Python Bridge (`bridge/`)

`libane_bridge.dylib` exposes a C-callable interface to ANE for use from Python or higher-level code. Build with `cd bridge && make`.

## Key Implementation Details

- **Process restart**: When `g_compile_count >= MAX_COMPILES (100)`, the process calls `munmap` + `close(data_fd)` then re-executes itself via `execl(realpath(argv[0]))` with `--resume`. The checkpoint is written before exec so training continues seamlessly.
- **OOM handling**: All `malloc`/`calloc` for float arrays go through `xmf(n)`/`xcf(n)` in `stories_config.h`, which call `abort()` with a diagnostic on NULL. Non-float allocations have inline `if (!p) { ... abort(); }` guards.
- **Checkpoint format**: `CkptHdr` struct with magic `0x424C5A54`, version 2. `pad[0] = 0x01020304` as little-endian sentinel (LE-only platform, enforced by `_Static_assert`). Old checkpoints (`pad[0] == 0`) are accepted.
- **SDPA causal masking**: ANE hardware ignores `attn_mask` in SDPA. Workaround: Q@K^T on ANE, add causal mask (CPU/ANE), softmax on ANE, scores@V on ANE.
- **Async gradient overlap**: `dW` gradients (`cblas_sgemm`) are dispatched to a serial GCD queue and waited on at the start of the *next* step's forward pass, overlapping with ANE evaluation.

## Repository Layout

```
api_exploration.m       ANE API discovery (standalone)
inmem_*.m / sram_*.m    Standalone ANE benchmarks
bridge/                 C-callable ANE bridge dylib
training/
  train_large.m         Static baseline main
  train_large_ane.m     Static + ANE extras main
  train.m               Early prototype (uses ane_runtime.h)
  training_dynamic/     Dynamic weight pipeline (self-contained)
  test_*.m              Per-feature kernel tests
  dashboard.py          TUI monitoring
  tokenize.py           BPE tokenizer -> .bin
  download_data.sh      TinyStories data download
docs/
  reports/              Security audit and analysis reports
  diaries/              Development session logs
```
