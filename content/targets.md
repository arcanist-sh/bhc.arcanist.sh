+++
title = "Target Backends"
description = "Compile Haskell to multiple targets"
template = "page.html"
+++

# Target Backends

BHC is designed for multiple targets. "Haskell" doesn't have to mean "one runtime on one OS."

## Available Targets

### Native

Server deployments and desktop applications.

```bash
bhc --target=native Main.hs
# or just:
bhc Main.hs
```

Status: **Supported**

- Full runtime features
- All profiles available
- Best performance for server workloads

### WASI/WASM

Sandboxed compute and edge deployments.

```bash
bhc --target=wasi Main.hs
```

Status: **Supported**

- WebAssembly output
- WASI system interface
- Works with edge profile
- Suitable for serverless and browser environments

Limitations:
- Some runtime features restricted by WASI
- Edge profile recommended for size

### GPU (Metal / CUDA / ROCm)

Accelerated compute for numeric workloads.

```bash
bhc --target=cuda Main.hs
bhc --target=rocm Main.hs
```

Status: **Metal runs on hardware; CUDA and ROCm are mock-validated**

- Apple Metal: the runtime compiles Metal Shading Language at run time and executes on the GPU of any Apple-silicon Mac. This is the one GPU path that runs, and is checked, on real hardware today. macOS only, behind the `metal` build feature; no `--target` flag yet.
- NVIDIA CUDA: PTX code generation, validated in mock mode; real-hardware testing pending
- AMD ROCm: AMDGCN code generation, structure complete; needs hardware
- Automatic kernel fusion from Tensor IR
- Device memory management
- Works with numeric profile

### RISC-V

Emerging architecture support.

```bash
bhc --target=riscv64 Main.hs
```

Status: **Supported**

- 32-bit and 64-bit RISC-V
- Native LLVM backend

### Experimental

Additional backends under consideration.

| Target | Status | Notes |
|--------|--------|-------|
| JavaScript | Planned | Browser-native execution |
| Bare metal | Experimental | No OS dependencies |

## Target Selection

### Command line

```bash
# Native (default)
bhc Main.hs

# WASI/WASM
bhc --target=wasi Main.hs

# Specify output
bhc --target=wasi -o app.wasm Main.hs
```

### Cross-compilation

BHC supports cross-compilation from your development machine to different targets.

```bash
# Compile WASM from macOS
bhc --target=wasi Main.hs -o app.wasm

# Run with wasmtime
wasmtime app.wasm
```

## Target-Specific Considerations

### Native Target

Full feature set:
- All runtime profiles
- Full FFI support
- Native threads and concurrency
- System calls unrestricted

### WASI Target

Restricted environment:
- WASI system interface only
- No native threads (cooperative concurrency)
- File system access through WASI capabilities
- Network access through WASI sockets (when available)

Recommended setup:
```bash
bhc --target=wasi --profile=edge Main.hs
```

### GPU Target

For numeric workloads with large data parallelism:

- Tensor IR operations automatically lowered to GPU kernels
- Device memory managed by runtime
- Host/device transfers optimized
- Kernel fusion reduces memory bandwidth

Recommended setup:
```bash
bhc --target=cuda --profile=numeric Main.hs
```

### FFI Considerations

Foreign function calls differ by target:

| Target | FFI Support |
|--------|-------------|
| Native | Full C FFI |
| WASI | WASI imports only |
| GPU | Device-side functions only |

For portable code, abstract FFI behind a platform layer.

## Runtime Size

Different targets have different runtime sizes:

| Target | Profile | Approximate Size |
|--------|---------|------------------|
| Native | default | ~2 MB |
| Native | server | ~2.5 MB |
| Native | numeric | ~2.2 MB |
| WASI | edge | ~500 KB |
| GPU | numeric | ~3 MB + driver |

Sizes are approximate and depend on what features your code uses. GPU targets require platform-specific drivers (CUDA toolkit or ROCm).

## Combining Targets and Profiles

| Profile | Native | WASI | GPU |
|---------|--------|------|-----|
| default | ✅ | ✅ | ❌ |
| server | ✅ | 🟡 (limited) | ❌ |
| numeric | ✅ | ✅ | ✅ (recommended) |
| edge | ✅ | ✅ (recommended) | ❌ |

The edge profile is designed for WASI and other constrained targets. GPU targets require the numeric profile for Tensor IR integration.
