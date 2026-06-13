---
layout: post
title: "Training a Geometric Language Model in Pure Rust: First Results"
date: 2026-06-12
categories: language-geometry
---

The [geometric decoder post](/language-geometry/2026/06/09/geometric-graph-attention-decoder.html) described how a corpus-native graph can guide token decoding through Rodrigues rotation and curvature weighting. This post covers what happens when you connect that graph to a training loop and actually try to learn next-token prediction from it.

Everything runs on CPU. No GPU, no autograd framework — just pure Rust with manual backprop.

---

## What's being trained

The architecture:

```
input:  8 context token positions × 3D coords = 24 floats
hidden: MLP(24 → hidden_dim → vocab_size)
output: softmax over dense vocab
```

The positions come from the same PMI+SVD pipeline as the decoder experiments: co-occurrence statistics → TruncatedSVD → unit-sphere 3D coordinates per token. The MLP maps those geometric coordinates to a next-token distribution.

What's not there: learned embeddings, attention, positional encoding, transformer blocks. The geometry is the representation. The MLP is the prediction head.

**Why no framework.** Burn and Candle both have poor CPU performance and are primarily CUDA infrastructure. The experiments are CPU-first and AMD GPU later. Writing the forward and backward passes directly in Rust costs a few hundred lines (`algorithms/mlp.rs`, `algorithms/adam.rs` in [geographdb-core](https://github.com/oldnordic/geographdb-core)) and avoids pulling in a dependency chain that doesn't fit the use case.

The backward pass for the Rodrigues layer:

```
δW_out  = h.T @ δlogits
δh      = δlogits @ W_out
```

Rodrigues rotation matrices are orthogonal, so `R^T = R^{-1}`. Gradients flow back through the transport step without inverting anything:

```
δh_u = Σ_{v: u∈N(v)} R_{vu}^T · δh2_v
```

Trainable parameters: the MLP weights only. The coordinate positions are fixed (frozen from the SVD), and the Rodrigues rotations have no parameters — they're computed from 3D positions at forward-pass time.

---

## Toy corpus: does the implementation work?

Before touching TinyStories, the training loop was tested on a hand-built two-community graph: 8 nodes split into two spatial clusters, sequences that walk within or between communities.

Result: **100% accuracy** after 200 epochs. Loss curve is monotonically decreasing. The MLP can learn to separate the two communities from 3D coordinate context alone.

This isn't impressive on its own — it's 8 nodes — but it validates that the forward pass, backward pass, Adam update, and the gradient accumulation are all correct.

---

## First run on TinyStories: a training bug

The first TinyStories run (2,000 stories, vocab 3,547 tokens + UNK, dim 64, lr=0.001) showed a diagnostic failure:

```
epoch 1  loss=7.50
epoch 2  loss=7.76
```

Loss went up on epoch 2. That's optimizer divergence, not architecture failure.

**Root cause:** the training loop was calling one Adam step per training example. Per-example Adam is stochastic gradient descent with maximum noise: each of the ~100K examples in a 2,000-story epoch produces its own independent parameter update, and Adam's moment estimates are meaningless when computed on a single data point. With a 3,547-class output, each update is 226K parameter changes computed from one token's gradient.

**Fix:** accumulate gradients over batches of 128, divide by batch size, then one Adam step. Standard mini-batch SGD. Lower default LR to 1e-4. Already had `clip_gradients` in `Adam` — wired it in.

With the fix, the same run:

```
epoch 1  loss=6.006
epoch 2  loss=5.831
epoch 3  loss=5.741
epoch 4  loss=5.668
epoch 5  loss=5.616
```

Monotonically decreasing across all 5 epochs. No divergence. Train loss ≈ validation perplexity (no overfitting — the model hasn't learned enough to overfit).

---

## Results: 2k stories, 5 epochs

| Model | Validation perplexity |
|-------|----------------------|
| Bigram (Laplace-smoothed) | 175.7 |
| Geometric MLP (frozen coords) | 282.8 |
| Geometric MLP + curvature weighting | 304.5 |

Bigram wins. The geometric model is learning (282 vs. ~4096 random), but not beating the baseline.

Two things are worth unpacking here.

**Why bigram wins.** Bigram takes the exact previous token as input and directly reads co-occurrence counts. The geometric MLP takes 3D positions as input. The SVD compression maps tokens to unit-sphere coordinates based on shared neighborhood structure — tokens that co-occur with similar neighbors end up nearby. But nearby tokens aren't identical: the 3D position is a lossy representation of the token identity. The MLP has to recover discriminative signal from compressed coordinates. Bigram has no such compression; it works directly from identity.

**Why the comparison is slightly asymmetric.** Bigram uses 1-token context. The geometric model uses 8-token context (8 × 3D positions). The geometric model has more information in principle, but at this data scale the 3D coordinates don't carry enough structure to exploit the longer context. With 2,000 training stories, the PMI co-occurrence matrix is sparse — many token pairs never co-occur, and the SVD positions don't reliably separate semantically distinct tokens.

**Why curvature weighting hurts.** The curvature evaluation adds a heuristic log-probability bias (angle continuity + κ penalty, both with fixed coefficients) on top of the learned MLP logits at inference time. If the MLP has already learned something useful, overlaying an untuned heuristic distorts it. The curvature signal isn't useless — it actively helped in the decoder traversal experiments — but there it was the only signal. Adding it as a fixed-coefficient bonus over a trained model requires tuning those coefficients, not hardcoding them at 1.0.

---

## 20,000 stories, 15 epochs: the full result

The 20k run added two variants not tested before: a **trigram** model (takes two previous token IDs, no geometry) and a **hybrid** model (two previous token IDs + 8 previous 3D positions). This makes the comparison direct: does geometry add anything on top of token identity?

| Model | Validation perplexity |
|-------|----------------------|
| Bigram baseline | 72.97 |
| Trigram (token identity only) | 32.02 |
| Hybrid (token identity + geometry) | 43.24 |
| Hybrid + κ weighting | 43.81 |

**Geometry does not add signal.** Trigram beats hybrid by 11 perplexity points. The MLP gets a cleaner signal from two token IDs than from two token IDs plus 8 × 3D coordinates. The curvature-weighted variant is slightly worse than plain hybrid.

Training dynamics match the numbers. Trigram fit the training set harder and plateaued around loss 3.18. Hybrid plateaued around 3.61 and started overfitting after epoch 9 — the geometric features are hurting generalisation, not helping it.

**Why geometry doesn't help here:**

PMI+SVD positions encode shared co-occurrence neighborhood structure. Tokens that appear in similar contexts end up nearby in 3D space. That's useful for finding semantically related tokens, but next-token prediction doesn't need semantically related tokens — it needs the likely *next* token given the current context. A 3D coordinate tells you what a token is *like*; it doesn't tell you what comes after it. The token ID tells you both.

The 8-position geometric context should in principle carry more information than a single token ID (which is what bigram uses). In practice, the MLP can't extract that signal from the SVD coordinates. The two-token-ID trigram dominates by a large margin over everything else.

---

## Geo-attention: single-head graph attention over geometric neighbors

The MLP result raised a different question: maybe the architecture is the constraint, not the representation. An MLP treats all 8 context positions equally and independently. A token's geometric neighbors might carry signal that only becomes useful when actively queried — matching what the current token is "looking for" against what its neighbors know.

`GraphAttentionClassifier` implements this directly:

- Token embedding table (learned)
- Learned W_q, W_k, W_v projections
- Each context token attends to itself + its k geometric neighbors from the PMI graph
- Residual update: `h = embedding + attention(...)`
- MLP head on the last context position
- Full backward pass through attention weights and MLP

The same 20k/15ep setup, four variants in parallel:

| Model | Validation perplexity |
|-------|----------------------|
| One-hot trigram (baseline) | 32.02 |
| Geo-attention + 4 neighbors | 55.54 |
| Geometric rotated + 4 neighbors | 126.85 |
| Geometric absolute | 145.08 |
| Geometric rotated (no neighbors) | 272.98 |

**Attention over geometry is much better than MLP over geometry.** Geo-attention (55.54) is roughly 2.5x better than the best MLP-on-geometry variant (127 ppl). The query/key/value mechanism gives the model a "search and correlate" capability the flat MLP doesn't have: it can weight neighbors selectively based on what the current token embedding is asking for.

**Geometry still loses to token identity.** Even with attention, geo-attention is 23 ppl behind one-hot trigram. Rotation alone (no neighbors) was near-useless (273 ppl); adding 4 neighbors rescued it to 127 ppl. Local geometric neighborhoods carry some signal — but only when actively queried, and not enough to close the gap with trigram.

**Why the gap persists.** PMI+SVD positions cluster tokens by shared co-occurrence context — tokens that appear in similar environments end up nearby in 3D space. That's a *semantic* similarity measure. Next-token prediction needs *successor* structure: which token tends to follow this one. These are different things. "Dog" and "cat" are geometric neighbors (similar contexts); neither predicts the other as a next token. The trigram baseline reads co-occurrence directly as successor frequency. The PMI graph doesn't preserve that direction.

---

## Where this leaves things

The full experiment arc so far, at 20k stories / 15 epochs:

| Architecture | Representation | Validation ppl |
|---|---|---|
| MLP | One-hot trigram | **32.02** |
| MLP | Hybrid (token ID + geometry) | 43.24 |
| Attention | Graph neighbors | 55.54 |
| MLP | Geometric rotated + neighbors | 126.85 |
| MLP | Geometric absolute | 145.08 |
| MLP | Geometric rotated | 272.98 |

The bottleneck is the PMI+SVD graph construction, not the model. To beat trigram with geometry, the geometric space itself needs to encode successor structure — either learned end-to-end, or derived from a graph that preserves directional co-occurrence rather than symmetric neighborhood similarity. That's the next question.

---

## Reproduce

```bash
git clone https://github.com/oldnordic/geographdb-core
git clone https://github.com/oldnordic/geographdb-experiments

cd geographdb-experiments
cargo run --release --bin train_geometric -- \
  --dataset roneneldan/TinyStories \
  --vocab-size 4096 \
  --dim 64 \
  --epochs 5 \
  --lr 1e-4 \
  --max-train-stories 2000 \
  --max-val-stories 1000
```

Hardware: AMD Ryzen 7 7800X3D, 64 GB RAM, no GPU used. Training 2k stories for 5 epochs takes roughly 8 minutes on this machine.

The tokenizer is cached to `--output` (default `/tmp/train_geometric_tinystories`) after the first run.

## Code

- MLP ops + backward: `geographdb-core/src/algorithms/mlp.rs`
- Adam optimizer: `geographdb-core/src/algorithms/adam.rs`
- Training binary: `geographdb-experiments/src/bin/train_geometric.rs`
- Rodrigues rotation: `geographdb-core/src/algorithms/parallel_transport.rs`
