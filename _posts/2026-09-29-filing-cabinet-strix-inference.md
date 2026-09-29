---
layout: post
title: "The filing cabinet lesson: running 125B on gaming hardware, and what it confirmed about our Strix campaign"
date: 2026-09-29
categories: llama-rs inference amd-strix qwen
---

![GPU homelab inference rig](/assets/images/image2art.jpg)

Andrew Zhu's recent write-up on Qwen3.8-Flash-Next — *"How I Ran The 177B Model on One Gaming GPU — and Why the Secret Isn't the GPU"* — is worth reading in full, but the core idea is simple enough to state in one sentence: the model splits its knowledge across the memory hierarchy by what each tier is good at, so the GPU stops being the library and becomes the reading room.

The architecture does four things that matter for anyone running inference on consumer silicon:

1. **Ultra-sparse MoE.** 512 experts per layer, 10 routed + 1 shared per token. Roughly 2% of the experts do the work, so "on call" experts can live in system RAM instead of VRAM.
2. **GDN + sparse attention.** Three of four layers compress history into a fixed-size state; the sparse-attention layer retrieves from a fixed budget regardless of context length. The KV cache stops scaling like a mansion and starts scaling like a sticky note.
3. **A gated, widened residual stream.** Four parallel lanes instead of one highway, with learned gates.
4. **The n-gram table — the filing cabinet.** A lookup table of phrase-level embeddings, bigger than the rest of the model, that performs zero matrix multiplications. Its addresses depend only on the input text, so the CPU can prefetch rows while the GPU is still busy. It lives in RAM, or even on SSD, and costs the GPU nothing.

Zhu's punchline is that this is why a 180B-class model fits on used RTX 3090s: VRAM holds the 6B that actually compute per token, RAM holds the experts and the table, NVMe holds the overflow. What I want to add here is what happened when we chased the same model on AMD Strix Halo (gfx1151) with our own Rust engine — because the filing-cabinet philosophy turned out to be exactly the right frame for interpreting our own measurements.

## Where llama-rs stands on gfx1151

All numbers below are measured on the same Strix Halo box, same model (Flash-Next, UD-IQ3_XXS), with canonical receipt checks (fixed prompts must produce bit-identical token sequences across runs) as a correctness gate. No number ships without receipts.

| metric | llama.cpp | llama-rs | verdict |
|---|---|---|---|
| tg128 decode | 19.92 t/s (tg1 anchor) | ~20.5 t/s | parity, slightly ahead |
| pp512 prefill | 226.0 ± 0.3 t/s | ~200.8 t/s (warm mean) | behind by ~11% |
| pp512 stacked | — | ~302 t/s | ahead (no llama.cpp equivalent) |

Two side notes that matter more than the table. First, hipEngine — the other engine targeting this silicon — cannot run this quantization on gfx1151 at all; the kernels simply aren't there. llama-rs is currently the only engine that runs UD-IQ3_XXS Flash-Next on this hardware at all. Second, decode parity was not free: it took ten-plus rounds of kernel surgery (bitonic top-k in shared memory, fused router + expert bins, in-warp activation quantization, partial HIP graph replay) to close from ~10 t/s to ~20.5 t/s. Dispatch overhead and memory roundtrips, not math, were the whole gap.

## The prefill gap is a library, not a loop

The most useful measurement of the last round was decomposing *where* llama.cpp still beats us on prefill. The answer maps cleanly onto the filing-cabinet idea:

- llama.cpp spends **1515 ms GPU-active** on the pp512 workload where we spend **2405 ms** — their dense GEMM kernels (F32, Q8_0, Q6_K) are simply faster, by +422 / +277 / +195 ms respectively.
- But llama.cpp pays **852 ms of host-side scheduling** where we pay **58 ms**. Our orchestration layer is fourteen times cheaper; their kernels are faster.

The gap is not architectural. We already have rocBLAS paths in-tree behind a flag; the next round promotes them to default for large-M GEMMs. If the estimate holds, prefill lands around 280 t/s — past llama.cpp, on top of the decode parity we already hold.

The lesson generalizes: on unified-memory silicon, the question is never "GPU or CPU" but "which bytes does each tier touch, and how often." Flash-Next's designers made that split inside the model; we have to make it inside the engine.

## The two tracks from here

**MTP (multi-token prediction).** Flash-Next ships a trained one-layer draft head — the `nextn` tensors (`eh_proj`, `enorm`/`hnorm`, shared head) are right there in the weights, and reference implementations show the shape: take the trunk's last hidden state plus the predicted token, fuse, run a single transformer block, verify speculatively. Unsloth reports 1.3–1.7× with no accuracy degradation. We have a golden-vector parity oracle in-tree and gated; the runner integration is queued behind the prefill work. On top of a 20 t/s base, that's the difference between "usable" and "pleasant."

**Recurrent depth — but at train time, not inference.** We ran a controlled probe on looping hidden states at inference time: feed the state back through layers, measure. The verdict was unambiguous and pre-registered — naive recurrence collapses (arithmetic accuracy fell from 40% to 1% at two loops), while a self-consistency control on the same harness improved (84% → 97%), proving the harness can detect a real effect when one exists. Conclusion: recurrent depth is a real capability — the Astra rumors and the open replications both point that way — but it belongs in the architecture at training time, not bolted onto a frozen model at decode. That's where it's going in our 2-bit model work.

## What the filing cabinet actually teaches

Zhu's article is nominally about one model on one GPU, but the transferable idea is a discipline: know which bytes are compute and which are lookup, and put each where they're cheap. Our campaign data says the same thing from the engine side — our wins came from keeping activations in shared memory and killing dispatches, our remaining gap is literally "their GEMM library reads bytes better," and the model itself hands us a 29GB lookup table that never needs the GPU at all.

The biggest models in the world are quietly designing themselves to fit on our machines. The engines that serve them have to finish the job.
