---
layout: post
title: "A Geometric Decoder for Language Graphs"
date: 2026-06-09
categories: language-geometry
---

This post is the decoder half of a larger experiment: can a language model decode by moving through a graph instead of comparing query/key vectors?

The standard transformer decoder scores every candidate token with `softmax(QK^T)`. The geometric alternative scores candidates by how well an outgoing edge continues the local curvature of the graph. No learned attention weights, no softmax over the full vocabulary, no feed-forward on hidden states. For this experiment, the graph acts as the decoding substrate.

## Mechanism

Three pieces are wired together in [`geographdb-core`](https://github.com/oldnordic/geographdb-core):

**D1 — Rodrigues rotation.** The walker carries a velocity vector along the incoming edge. To score a candidate outgoing edge, it rotates that velocity onto the candidate direction using Rodrigues' formula. The rotation angle is the cost: a straight continuation scores high, a sharp turn scores low.

**D2 — Graph attention + UCB.** For each candidate neighbor, the attention score is the cosine similarity of the *transported* neighbor feature against the current node's feature. `W_Q` and `W_K` are identity for this first test, so the geometry is doing all the work. An upper-confidence-bound term adds exploration pressure based on how often each node has been visited.

**E6 — Ricci curvature κ.** The UCB bonus is weighted by `1 - κ`, where κ comes from a fast Dice neighbor-overlap approximation. Negative κ means the edge is a bridge between communities; positive κ means it sits inside a tight cluster. The walker therefore explores more aggressively across bridges and less inside echo chambers.

## Metrics

Two simple counts:

- `unique-node coverage rate = unique_nodes / steps`. Higher means the walker visits more distinct nodes; lower means it loops.
- `cross-community` transitions: how often the walker moves from one side of the graph to the other.

Interpreting the two together matters. A walker can achieve high coverage by bouncing around inside one community without ever leaving. The geometric walker deliberately trades some local coverage for cross-community coherence.

## Toy graph

A 6-node graph with two clusters:

| Mode | Unique nodes | Coverage rate | Cross-community |
|------|--------------|---------------|-----------------|
| Distance-KNN | 2 | 0.065 | 0 |
| Graph attention + UCB | 6 | 0.194 | 3 |
| Graph attention + UCB + κ | 6 | 0.194 | **7** |

Distance-KNN gets stuck between the two nearest nodes and never crosses communities. The geometric walker visits every node, and κ doubles the number of community crossings.

## Corpus-scale: full TinyStories

The same comparison on the full TinyStories training set:

- 2,119,719 texts
- 460,035,016 tokens
- 43,047 vocabulary tokens → 54,919 sense-nodes
- 859,917 PMI edges

| Mode | Unique nodes | Coverage rate | Cross-community |
|------|--------------|---------------|-----------------|
| Distance-KNN | 397 | 0.792 | 62 |
| Graph attention + UCB | 284 | 0.567 | 186 |
| Graph attention + UCB + κ | **304** | 0.607 | **247** |

Distance-KNN has the widest surface coverage (79% new nodes) but shallow semantic reach: only 62 cross-community moves. The geometric walker visits fewer unique nodes overall but makes 3× more cross-community transitions. Adding κ pushes cross-community movement another +33%.

## Ablation readout

```python
import numpy as np

knn = np.array([397, 62])
attn = np.array([284, 186])
ricci = np.array([304, 247])

print("cross-community gain over Distance-KNN:")
print(f"  attention:   {attn[1] / knn[1] - 1:.2%}")
print(f"  attention+κ: {ricci[1] / knn[1] - 1:.2%}")
print(f"  κ-only lift: {ricci[1] / attn[1] - 1:.2%}")
```

Output:

```text
cross-community gain over Distance-KNN:
  attention:   +200.00%
  attention+κ: +298.39%
  κ-only lift: +32.80%
```

The gain from κ is additive, not redundant: it specifically increases the fraction of moves that cross semantic boundaries.

## What this does and does not show

This shows that a geometric decoder can traverse a corpus-native graph more coherently than a pure distance heuristic, and that curvature is a useful signal for where to explore. It does **not** show that this decoder beats a trained transformer on perplexity or downstream tasks. The comparison is against a strong-but-myopic baseline, not against a full language model.

## Reproduce

Build the graph:

```bash
cargo run --release --bin corpus_native_graph -- \
  --dataset text=roneneldan/TinyStories:data/train- \
  --vocab-size 50000 \
  --output /tmp/tinystories_full_graph
```

Run the three-way comparison:

```bash
cargo run --release --bin d3_attention_comparison -- \
  /tmp/tinystories_full_graph 500 0.0
```

The `0.0` is the edge-weight threshold for the fast κ computation; TinyStories edges are PMI-weighted, so zero keeps all of them.

## Code

- D1/D2: `geographdb-core/src/algorithms/parallel_transport.rs`, `geographdb-core/src/corpus/inference.rs`
- D3: `geographdb-experiments/src/bin/d3_attention_comparison.rs`
- E6: `geographdb-core/src/algorithms/ricci.rs` (`curvature_map_fast`)

## Encoder note

The decoder experiments above assume the node positions are already meaningful. A first encoder test (E2) trains a small MLP, `CoordinateBranch`, to predict Memoria's layer-23 PCA activation coordinates directly from token IDs.

Those PCA coordinates are **unnormalized**:

| Stat | Value |
|------|-------|
| Per-axis std | `[9.55, 7.62, 6.20]` |
| Norm std (‖σ‖) | **13.70** |
| Mean norm (‖x‖) | **12.06** |

After 200 epochs the model reaches **mean L2 error = 0.519**. In context:

- `0.519 / 13.70 ≈ 3.8%` of the coordinate spread
- **92.6%** of nodes are within L2 error < 1.0
- **99.99%** are within L2 error < 2.0

That says PMI co-occurrence geometry captures the **rough layout** of the activation manifold, but not the fine-grained positions. It is a meaningful correlation, not a pixel-perfect reconstruction.

## Next

A full encoder-decoder post will connect the CoordinateBranch encoder (positions from text and from measured activations) back to the geometric decoder above.
