---
layout: post
title: "Ollivier-Ricci Curvature in a Corpus-Native Graph — No Training Required"
date: 2026-06-11
categories: language-geometry
---

Kuo et al. (Yale 2024) measured Ollivier-Ricci curvature in trained transformer embeddings (Llama 2, Gemma 2, DeepSeek) and found substantial negative mean curvature — the geometry of the embedding space is hyperbolic, not flat. Their interpretation: the structure emerges from training.

I wanted to test a different hypothesis: the curvature is in the language itself, not the training. If a graph built from raw co-occurrence statistics — no neural network, no GGUF, no weights — shows the same signature, then the geometry is a property of language topology.

## What I built

Pipeline:
1. TinyStories dataset (English children's stories, ~2M tokens after tokenization)
2. GPT-2 tokenizer, skip-gram PMI co-occurrence matrix
3. TruncatedSVD → 3D token positions (much faster than full SVD: 0.1s vs 237s)
4. NetworkX graph from strong bigrams (count ≥ 10), average degree ~15.6
5. Louvain community detection (resolution=3.0 → 211 communities)
6. Ollivier-Ricci curvature via neighbor-overlap Dice approximation

The Dice formula for κ(u, v):

```
κ(u,v) = 2 * |N(u) ∩ N(v)| / (|N(u)| + |N(v)|) - 1
```

This is position-independent — it measures how much the neighborhoods of two tokens overlap. Intra-community edges (tokens that share many neighbors) tend toward κ → 0 or positive. Inter-community bridges (tokens connecting otherwise-separate clusters) tend toward κ < 0.

## Results

2,000 intra-community edges sampled, 2,000 inter-community edges sampled.

| Metric | Value |
|--------|-------|
| Communities detected | 211 |
| Intra-community mean κ | −0.899 |
| Inter-community mean κ | −0.907 |
| Overall mean κ | −0.903 |
| % positive κ | 0% |

Negative mean curvature: confirmed. No training involved.

Walk traces from common seed tokens ("the", "once", "a", "he") show κ staying consistently negative across 30-step walks, with values clustering in the −0.7 to −1.0 range.

## What didn't work

The Yale paper shows a **bimodal** distribution: intra-community edges with κ > 0 (spherical regime, tight semantic clusters) and inter-community edges with κ < 0 (hyperbolic regime, bridging between domains). I expected to reproduce that.

I didn't. Both intra and inter community edges show similar strongly negative curvature.

The reason is corpus homogeneity. TinyStories is a single-domain corpus — children's stories with simple vocabulary and repetitive structure. The co-occurrence graph is dense and uniform. There are no sharp semantic boundaries between "physics vocabulary" and "cooking vocabulary" because that variation doesn't exist in the data. Louvain finds 211 communities, but they're not semantically distant enough to show the bimodal split.

First attempt also failed differently: using the centroid approximation (W1 lower bound via 3D SVD positions), TruncatedSVD explained only 2.5% of variance in 3 dimensions. After unit-sphere normalization, all centroids averaged near the sphere center, producing artificially high (and incorrect) curvature estimates. Switched to the position-independent Dice formula after that.

## What this means

Negative mean curvature appears in a corpus-native graph with no training. The geometry is at least partially a property of language co-occurrence structure, not exclusively a consequence of gradient descent.

This is consistent with the Yale result but doesn't reproduce the full bimodal signature. To test that, the corpus needs semantic heterogeneity — Wikipedia or a multi-domain mixture where topic clusters are genuinely distinct.

## Code

`scripts/experiment_curvature.py` in the [Memoria repository](https://github.com/oldnordic/Memoria). Requires NetworkX, scikit-learn, HuggingFace datasets, geographdb (local).

Elapsed time on Ryzen 7 7800X3D: 30.8 seconds.

## How to reproduce

```bash
git clone https://github.com/oldnordic/Memoria
cd Memoria
pip install datasets transformers scikit-learn networkx matplotlib scipy
python scripts/experiment_curvature.py
# outputs: data/curvature_results.json, data/curvature_kappa_dist.png
```

Hardware used: AMD Ryzen 7 7800X3D, 64 GB RAM, no GPU required. Elapsed: 30.8 seconds.

The script downloads TinyStories (~2 GB) on first run via HuggingFace datasets. `geographdb` is a local dependency from the same repo — import path is patched in the script header.

## References

- Kuo et al., "Geometry of Transformer Embeddings" (Yale 2024) — curvature measurements in trained LLMs this experiment responds to
- Ollivier, "Ricci curvature of Markov chains on metric spaces" (2009) — original definition
