---
layout: post
title: "Geometric-Only Attention: Linear Scaling from Sparse Neighborhoods"
date: 2026-06-14
categories: language-geometry
---

The [previous posts](/language-geometry/2026/06/13/geometry-as-substrate.html) documented a ceiling: every geometric attention variant converged to ~50.5 val perplexity on TinyStories, while a plain trigram MLP reached 32. Static geometry was the bottleneck.

This post covers two things: a new sparse attention mode in `geographdb-core` 0.5.3, and the first training run that goes below that ceiling.

---

## Geometric-only attention

`geographdb-core` 0.5.3 adds `GraphAttentionClassifier::set_geometric_attention_only(bool)`. When set:

- Each token attends only to itself and its geometric graph neighbors (fixed count `k`)
- The O(L²) full self-attention term is dropped entirely
- Complexity becomes O(L × k), where k is constant

The implementation is a sparse index build (`build_attended_indices`) shared between forward and backward pass. Softmax and RoPE are dispatched to CPU-native kernels with AVX2 where available, scalar fallback elsewhere.

---

## Benchmark

Measured on CPU, forward pass, context lengths L ∈ {8, 16, 32, 64, 128}. Geometric-only mode only goes to L=128 in this run; hybrid stops at L=64 because the quadratic cost makes longer sequences prohibitive in the benchmark setup.

| L   | Hybrid (default) | Geometric-only | Speedup |
|-----|------------------|----------------|---------|
| 8   | 122 µs           | 57 µs          | ~2.1×   |
| 16  | 390 µs           | 110 µs         | ~3.5×   |
| 32  | 1.38 ms          | 219 µs         | ~6.3×   |
| 64  | 5.13 ms          | 436 µs         | ~11.8×  |
| 128 | —                | 865 µs         | —       |

The speedup grows with L because the hybrid cost grows as L², while geometric-only grows as L. At L=64 it's ~12×; at L=128 the hybrid isn't benchmarked but would project to ~20 ms based on the quadratic trend.

These are wall-clock times, not theoretical FLOPs. AVX2 dispatch affects both modes so the ratio is a fair comparison.

---

## What this costs in accuracy

Unknown yet. The benchmark measures speed; it doesn't measure whether the geometric-only output is close to the hybrid output. That comparison is the next measurement. The sparse mode is a strict approximation of the full attention — it attends to fewer tokens — so output divergence is expected. How much, and whether it correlates with perplexity loss, is not yet measured.

---

## The training run

Separately, a cross-context attention variant is training on TinyStories (20k train / 2k val, 10 epochs). At the time of writing it is on epoch 3. Numbers so far:

| | Val perplexity |
|--|--|
| Bigram baseline | 115.300 |
| Epoch 1 | 34.527 |
| Epoch 2 | 31.607 |

For reference, all prior geometric attention variants — PMI substrate, transition-matrix substrate, with and without RoPE, with and without curvature weighting — converged in the range 50.4–50.5 and did not go lower. The trigram MLP baseline is 32.02.

Epoch 2 of this run is 31.607. That is below the prior ceiling and below the trigram baseline.

This is a single run, not yet complete. Early-stopping patience is 2 epochs. The model may still plateau or overfit. The architecture change that produced this is cross-context attention on top of geometric positions — the model can now query global context rather than being restricted to local neighborhood structure.

Whether it holds through epoch 10, and what the final number is, will be in a follow-up post.

---

## Connection between the two

Geometric-only mode is a sparse approximation to full attention. The benchmark above shows the cost reduction. The training run shows a model that needs cross-context (full) attention to break the 50-ppl ceiling — which means the full O(L²) term carries information the geometric neighborhood alone does not.

That is the tradeoff: geometric-only is fast and scales linearly, but (based on current evidence) loses the cross-context signal that closed the gap with the trigram baseline. Whether a hybrid strategy — geometric-only for most tokens, full attention on a small selected subset — can recover accuracy at lower cost is an open question and not yet measured.

---

## Code

`geographdb-core` 0.5.3 is at [github.com/oldnordic/geographdb-core](https://github.com/oldnordic/geographdb-core). The geometric-only flag, benchmark, and new test (`learns_simple_task_geometric_only`) are in this commit.
