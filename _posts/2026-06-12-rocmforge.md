---
layout: post
title: "rocmforge: Building an AMD GPU Inference Engine from Scratch"
date: 2026-06-12
categories: engineering
---

**This is a prototype.** It produces real throughput numbers on real hardware, it has a real GPU safety protocol because a real GPU page fault happened, and it implements a real stack of inference optimizations. It is also not finished, not stable, and not safe to run without caution. The CLAUDE.md for the project says: *"Before ANY new GPU code: Acquire cross-process GPU lock → Run staged preflight → Use timeout-wrapped subprocess."* That rule exists because ignoring it caused a desktop freeze.

With that said: here is what exists and why.

---

## The problem

Most open-source LLM inference runs on NVIDIA. The CUDA toolchain is deep, the community is large, and the optimizations compound. The AMD side — ROCm, HIP — is underserved. The existing tools (llama.cpp has ROCm support, but it's a port, not a first-class target) work well enough for basic use but don't expose the full AMD hardware stack.

I run an RX 7900 XT (gfx1100, RDNA3, 20GB VRAM). The specific goal I'm working toward: running Qwen3.6 with 256K context in 20GB VRAM without quality collapse. That requires KV cache compression, weight compression, and full AMD-native kernel paths. No CUDA compatibility layers.

[rocmforge](https://github.com/oldnordic) is the experiment.

---

## Architecture

Pure Rust + HIP. No Python in hot paths. No CUDA abstractions. AMD only.

```
src/
  gpu/           — GPU inference path (HIP)
    kernels/     — quantized GEMV/GEMM, attention, TurboQuant, flash attention
    forward/     — layer forward passes (decode, prefill, MoE, SSM)
    cache/       — KV cache: paged, TurboQuant compressed, prefix cache
    weights/     — GGUF + RFM weight loading, GPU upload
  cpu/           — CPU fallback path
  api/           — OpenAI-compatible HTTP server (axum/tokio)
  bin/
    convert.rs   — GGUF → RFM converter (SVD, MPO, TurboQuant)
```

The custom weight format is `.rfm` (ROCmForge Model). It stores compressed expert weights (SVD+sparse or MPO), KV centroid tables, and quantization metadata alongside the standard GGUF tensors.

---

## Throughput

Measured on RX 7900 XT, ROCm 7.2.0:

- **~515 tok/s decode** — Qwen2.5-0.5B Q4_0 (HIP graph capture path)
- **~106 tok/s decode** — Qwen2.5-7B Q4_0 (graph + Q8 fastpath)

These are real numbers from the profiling harnesses in `benches/`. They are also the easiest models to run — 0.5B fits trivially in VRAM, and 7B leaves plenty of headroom. The interesting work is everything else.

---

## TurboQuant KV cache compression

The standard KV cache layout stores full-precision K and V tensors per token. At 256K context on a 7B model, that's the entire VRAM budget. TurboQuant compresses each KV vector:

1. **FWHT pre-rotation** — Walsh-Hadamard transform spreads outliers before quantization
2. **Lloyd-Max quantization** — optimal scalar quantization for 3-bit (or 1/2/4-bit) centroid lookup
3. **QJL (1-bit)** — 1-bit sign projection for V cache

Layout per position in K cache:
```
| pack_bytes (centroid indices) | qjl_bytes (1-bit signs) |
```

Layout per position in V cache:
```
| pack_bytes (centroid indices) | 8 bytes RMS scales |
```

The per-position stride is `pack_bytes + max(qjl_bytes, TURBOQUANT_RMS_SCALE_BYTES)`. This formula took one bug to discover — the kernel used `max(pack_bytes + qjl_bytes, 8)` while the host used `pack_bytes + max(qjl_bytes, 8)`. They diverge when `pack_bytes > 0` and `qjl_bytes < 8`. That bug is now fixed with shared constants across the HIP kernel and Rust host code.

The claimed compression is ~4x VRAM reduction with L∞ ≤ 10⁻⁵ attention score parity. That claim holds when the source model weights are high-precision (F32, F16, or Q8_0). Using a low-precision GGUF (Q4_0 base) stacks quantization noise multiplicatively and degrades the result. The architecture invariant document says: *"For ultra-low-bit KV Cache compression to achieve near-lossless attention retrieval, the source model's weights in the GGUF must be in high precision."*

---

## FWHT+SVD weight compression

MoE expert weights in Qwen3.6 are large. For each expert layer, the approach is:

1. SVD decompose the weight matrix (gate, up, down projections)
2. Keep rank-k approximation (U, Σ, Vᵀ)
3. Quantize the sparse residual as CSR

When the sparse residual density exceeds 6.25% (the threshold where CSR stops paying off), fall back to MPO (Matrix Product Operator) decomposition as an alternative representation.

The GPU SVD uses rocSOLVER — measured at 100–500x faster than CPU power iteration depending on matrix size. The converter `rocmforge-convert` takes a GGUF, produces an `.rfm` with SVD+sparse or MPO payloads per expert.

The reconstruction error verified to **0.000000** against the source GGUF semantics. That's exact numerical equivalence for the SVD pass. The MPO path has a nonzero approximation error (it's lossy by design) — I haven't published a number for that.

---

## What else is implemented

The CHANGELOG covers all of this in detail. The short list:

**KV cache:**
- Paged KV cache — 16-token blocks, BlockAllocator with free-list + refcounting, copy-on-write for parallel sampling. Reduces waste from ~60% to <4%.
- Prefix cache — hashed token sequences, LRU eviction. Up to 5x TTFT reduction when system prompts repeat across requests.

**Serving:**
- OpenAI-compatible HTTP server (axum/tokio): `/v1/completions`, `/v1/chat/completions` (SSE streaming), `/v1/messages` (Anthropic API), `/v1/models/load`, `/v1/vram`.
- Per-model request serialization — one semaphore permit per model, prevents KV cache races.
- GPU device caching — `OnceLock<GpuDevice>`, eliminates per-request HIP stream initialization.
- GPU weight caching — `Arc<GpuModelWeights>` in `ModelEntry`, eliminates ~4GB per-request GPU reload.

**Batching:**
- Continuous batching (INF-12) — `DecodeBatch` slot table, configurable N concurrent sequences.
- Chunked prefill (INF-13) — 512-token default chunk size, prevents head-of-line blocking from long prompts. Slots transition from `Prefilling` to `Decoding` automatically.

**Kernels:**
- Native Q2_K/Q3_K GPU GEMV (no CPU fallback).
- Q4_K, Q5_K, Q6_K GEMV and GEMM dispatch.
- HIP graph capture with fast update (`hipGraphExecUpdate`) — avoids `hipGraphInstantiate` overhead on cache key mismatches.

**Speculative decoding** is wired into the server endpoints. A draft model can be loaded via `POST /v1/models/load` with `draft_model` parameter.

---

## The safety protocol

A real `amdgpu` page fault happened during prefill on an untested code path. The GPU reset. The desktop froze.

The current safety protocol:

1. **Cross-process GPU lock** — `flock`-based `GpuLock` with configurable timeout. Serializes across test processes.
2. **Staged preflight** — `gpu_safety_preflight()`: render node present → HIP sees device → memory roundtrip → trivial elementwise kernel. If any stage fails, stop.
3. **Desktop VRAM reservation** — `ROCMFORGE_DESKTOP_VRAM_GB=4.0` default. VRAM budget calculations subtract this before allocating inference memory.

The safety harness is in `src/gpu/safety.rs` and is called from `main.rs` and `tests/common/mod.rs`. Running GPU code without it is `unsafe` in the practical sense: it may work, it may cause a driver hang.

Build rule: `cargo build --features gpu` calls hipcc, which uses the GPU driver for code generation. Running this while Ollama has a model loaded caused a desktop freeze on 2026-05-31.

---

## Current state

The codebase is now fully modularized — no source file exceeds 1,000 lines, a cleanup pass completed recently (the CHANGELOG lists 9 modularization slices). GitHub Actions CI is wired up: CPU build/test, GPU `cargo check` inside a `rocm/dev-ubuntu-22.04` container, `cargo audit`, `cargo deny`, `gitleaks`, `semgrep`.

What works: CPU inference path runs reliably. GPU inference on Qwen2.5-0.5B/7B Q4_0 produces real throughput numbers. The HTTP server builds and responds to requests. TurboQuant KV compression builds and the layout math is correct.

What is not finished: the Qwen3.6 256K context target has not been run end-to-end. FWHT+SVD compression reduces MoE expert VRAM, but whether the full model fits in 20GB at 256K context with acceptable quality is an open question. The MPO path has not been benchmarked for reconstruction quality. Speculative decoding has server plumbing but no published throughput comparison.

The project is open-source at [github.com/oldnordic](https://github.com/oldnordic). Don't run the GPU path without reading the safety protocol first.
