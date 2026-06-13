---
layout: post
title: "Geometry as Substrate: What the Failing Results Are Telling Us"
date: 2026-06-13
categories: language-geometry
---

The [previous post](/language-geometry/2026/06/12/geometric-lm-training.html) documented a series of negative results: PMI+SVD geometric positions don't beat a trigram baseline, attention over geometry doesn't beat a trigram baseline, RoPE doesn't help, curvature weighting consistently hurts. Every experiment lost to two token IDs fed into a flat MLP.

This post is about what those results mean — and where the geometry idea goes from here.

---

## Negative results are signals, not failures

The full experiment arc so far, all at 20k TinyStories / 15 epochs:

| Architecture | Representation | Val perplexity |
|---|---|---|
| MLP | One-hot trigram | **32.02** |
| Attention | Graph neighbors (PMI) | 55.54 |
| MLP | Hybrid (token ID + geometry) | 43.24 |
| MLP | Geometric rotated + neighbors | 126.85 |
| MLP | Geometric absolute | 145.08 |
| MLP | Geometric rotated | 272.98 |
| Bigram baseline | — | 72.97 |

Read as failures: every geometric model lost.

Read as signals:

1. **Attention over geometry is 2.5x better than MLP over geometry** (55 vs 127 ppl). The architecture matters. Query/key/value over graph neighbors extracts real structure that a flat MLP can't see.

2. **Geometric models peak earlier than trigram.** Geo-attention trained to its best validation around epoch 2-3. Trigram kept improving through epoch 15. The geometry finds *something* fast — it just can't go as deep as direct token identity.

3. **Adding geometry to token identity hurts** (hybrid 43 vs trigram 32). The PMI positions are not just uninformative — they're noise on top of the token signal.

4. **Rotation alone is catastrophic; neighbors rescue it** (273 vs 127 ppl). Local geometric context carries signal, but only when the model can query it selectively.

The consistent pattern: the PMI+SVD substrate has *some* structure (early peaking, attention extractability) but the wrong kind of structure for next-token prediction. PMI encodes symmetric co-occurrence similarity. Next-token prediction needs directed successor structure. Those are different things.

---

## Why PMI is the wrong map

PMI+SVD clusters tokens by shared context neighborhood. "Dog" and "cat" end up near each other because they appear in similar sentences. Neither predicts the other as a next token. The map encodes *what is similar* — it doesn't encode *what comes next*.

Language has both. Words that are semantically similar (similarity structure) and words that tend to follow each other (succession structure). Current LLMs learn succession directly from token sequences. The PMI graph captures similarity and ignores succession.

The next experiment: swap PMI for a **directed transition matrix**. Build `P(w₂ | w₁)` from bigram counts, embed its spectral structure in 3D. Positions now encode "where does this token's probability mass flow to" — successor structure, not similarity structure. Same geo-attention architecture, different substrate. If the gap closes, directionality was the missing piece.

---

## The deeper problem: static geometry

Even with the right directionality, there's a harder constraint: the geometry is fixed at training time. PMI positions are computed from the corpus, frozen, and never updated. The model learns to read a static map.

Brains don't work this way. Synaptic weights update continuously from prediction outcomes. Connections that contribute to correct predictions strengthen. Connections that lead to errors weaken. The geometry itself is the thing being trained — not just the weights on top of it.

The current architecture has:
- Fixed graph topology (PMI-derived)
- Fixed node positions (SVD coordinates)
- Learned attention weights (W_q, W_k, W_v)
- Learned MLP head

The learned pieces are layered on top of a frozen substrate. The question the experiments are really answering is: *how much can learned attention compensate for a wrong substrate?* The answer so far: partially (55 vs 127 ppl) but not enough (55 vs 32 ppl).

**Learned graph plasticity** is the next structural change. Make the edge weights learnable. Gradient flows back through the attention mechanism and updates not just the attention projections but the graph connectivity itself. The topology stays fixed (PMI-derived initial graph) but edges strengthen or weaken from prediction signal.

Over training, edges that helped predict correct next tokens survive. Edges that didn't, decay. The graph self-organizes from co-occurrence structure toward successor structure — the same information the trigram uses directly, but learned geometrically rather than counted statistically.

This is Hebbian plasticity: neurons that fire together wire together. In the graph: paths that correctly predicted the next token get reinforced. The geometry evolves to encode what the task needs.

---

## Three connected ideas

The experiments are testing stage one of a larger architecture. The three ideas are connected:

**Geometry** is the substrate — the space where tokens live and where computation happens. Not flat embedding space (which transformers use), but a space with native structure: distance, direction, curvature, neighborhood.

**Multi-sense tokens (quantum-token)** is the representation — each token is not a point but a distribution over possible states. The same token "bank" in different geometric neighborhoods activates different sense vectors. Context collapses the superposition. This is what attention approximates, but with fixed weights and no geometric grounding. A token whose local curvature is high is near a boundary between senses — geometrically ambiguous.

**Plasticity** is the learning rule — edge weights update from prediction outcomes, the geometry self-organizes toward the task. Not backprop through frozen structure, but backprop that changes the structure itself.

Transformers have a version of each: attention approximates multi-sense (context-dependent activations), positional encodings inject weak geometry, and gradient descent updates the weights. But the geometry is not native — it's injected as a correction to an orderless token bag. And the weights freeze after training. No online adaptation, no structural update.

---

## Fractals, attractors, and deterministic structure

Current LLMs are statistical approximators. They learn to predict what *usually* comes next from training data. They guess from patterns.

Language has deterministic structure underneath the statistics. Grammar rules are recursive — sentences contain sentences, phrases contain phrases, at every scale the same rules apply. Mathematical proofs are deterministic — the same axioms always produce the same theorems. Code is exact. Even narrative has deep structure (the same story morphology appears across cultures and languages independently).

Fractals are the extreme case: one formula, infinite complexity, perfectly deterministic. `z = z² + c` generates the Mandelbrot set. Zoom into any boundary region and the same structure appears, because the same rule is being applied. Nature uses this everywhere — tree branching, leaf venation, vascular networks, coastlines — because recursive self-similar geometry is how you pack maximum function into minimum description.

The implication for language geometry: if concepts have **geometric attractors** — regions of the token space that the dynamics always flows toward given certain inputs — then reasoning is navigation, not guessing. The geometry carries the generative rule. The forward pass follows it.

This is speculative. But the tensor field added to `geographdb-core` is a step toward it. Local curvature tensors encode how the space bends around each token — how strongly the geometry pulls toward an attractor in that region. High curvature = strong rule = low ambiguity. Low curvature = flat region = multiple paths equally likely.

If that curvature signal can be incorporated as a per-token feature — alongside position, direction, and neighborhood — the geometry starts to encode not just *where tokens live* but *how strongly the local rules constrain what comes next*.

---

## Where this is going

The immediate next experiment: **directed transition matrix substrate**. One variable changes — PMI positions swap for transition-spectral positions. Same geo-attention model. If the gap with trigram closes, directionality was the bottleneck and the architecture is sound.

If the gap doesn't close: the problem is static geometry regardless of directionality, and **learned edge plasticity** becomes the next experiment. Start from a dense k-NN initial graph, make edge weights learnable, train with the same gradient that updates attention weights.

Longer term:

- Tensor curvature as per-token input feature
- Multi-sense token geometry (mixture of position distributions, context-selected)
- Online plasticity: edge weights that update at inference time from context, not just during training

None of these alone is a new transformer. Together, they're a different kind of substrate — one where structure is native rather than approximated, and where the geometry itself carries information that statistics would need much more data to recover.

The failing results are pointing at what the substrate is missing. That's exactly what experiments are for.
