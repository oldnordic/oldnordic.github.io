---
layout: post
title: "Multi-Layer Graphs, Ricci Curvature, and a Hypothesis About How Computation Should Route"
date: 2026-06-15
categories: language-geometry
---

This post is about an idea, not a result. The experiment described here has not been run. The hypothesis may be wrong. I'm writing it down because the reasoning is worth making explicit before touching code.

---

## The transformer's structural problem

A transformer collapses all abstraction levels into one operation. Syntactic prediction, semantic disambiguation, logical inference, meta-reasoning — the same matrix multiply handles all of them. The "knowledge" is distributed across billions of parameters with no structural distinction between "this weight encodes grammar" and "this weight encodes logical entailment."

This creates two practical problems:

**The frozen state problem.** Weights are fixed after training. To update what the model knows, retrain everything. There's no mechanism for local update — no way to say "strengthen this specific connection because it produced a correct prediction."

**The cost problem.** A matrix multiply over a 50k vocabulary doesn't distinguish between an unambiguous token ("the" after "in") and a highly ambiguous one ("bank" after "I went to the"). Both pay the same computational cost. The operation is uniform where the problem is not.

Research on Ollivier-Ricci curvature in transformers shows that the geometry is already there. Attention heads develop curvature concentration at semantically load-bearing positions — some edges carry most of the semantic weight. The structure is real; it's just hidden inside the matrix where it can't be used for routing.

---

## A multi-layer graph alternative

The hypothesis: replace dense matrix computation with a layered sparse graph where each layer handles one abstraction level, and a tensor field of Ricci curvature determines where layers naturally couple.

Each layer is a sparse graph with its own nodes, edges, and responsibility:

```
Layer 4: Meta — reasoning about reasoning, past trace annotations
Layer 3: Logical / causal — entailment, causality, succession
Layer 2: Semantic / conceptual — similarity, sense disambiguation  
Layer 1: Syntactic / token — co-occurrence, raw sequence statistics
```

Each layer produces a partial result. The output is a weighted sum:

```
output = Σ(layer_i_result × inter_layer_weight_i)
```

The inter-layer weights are graph edges, not learned gates. Sparse, inspectable, updatable from prediction outcomes without retraining.

Cost scales with active connections, not with vocabulary size squared. An unambiguous token may require only Layer 1. An ambiguous one — high local curvature — pulls weight from Layer 2 upward.

Adding a new layer means adding new edges into the sum. Existing layers are unchanged.

---

## The tensor as routing signal

In Riemannian geometry, the metric tensor encodes how space curves. Curvature tells you whether a path between two points bends or goes straight, which determines which paths are actually short.

The hypothesis applies the same idea to the multi-layer graph: **Ricci curvature at an inter-layer edge is the routing weight.**

Where the tensor bends sharply — high local curvature — layers are pulled together. Inter-layer edges activate, computation lifts to the next abstraction level. Where it's flat, layers don't interact.

The analogy to General Relativity: matter tells spacetime how to curve, curvature tells matter how to move. Here: information density in the graph tells the tensor how to curve, curvature tells computation where to flow between layers.

High curvature = concept boundary = ambiguity = lift to higher layer. Low curvature = unambiguous region = stay at current layer.

This is not a new routing mechanism bolted on. It's the geometry of the graph itself, expressed as a curvature field, doing the routing.

---

## Why observe before building

The previous geographdb experiments produced a clear lesson: don't impose structure, measure it.

The PMI substrate experiments (50.4–50.5 ppl ceiling) showed that geometry imposed from co-occurrence statistics doesn't transfer to next-token prediction. The geometry had to be the *right kind* for the task. Imposing the wrong geometry was worse than no geometry.

The same lesson applies here. If the curvature field is imposed — "high-curvature nodes connect to Layer 2, low-curvature to Layer 1" — that's a design decision that may or may not match what the data actually needs.

The right approach, following the Ollivier-Ricci methodology: build the multi-layer graph from existing data, compute curvature on every edge (within layers AND inter-layer), and observe where it concentrates.

The null hypothesis: curvature distributes randomly across layers and inter-layer edges. If high-curvature points in Layer 1 don't align with high-curvature points in Layer 2 at the same concept, the inter-layer routing hypothesis is wrong.

The signal to look for: if curvature aligns across layers at the same concepts without being forced — if the geometry self-organizes the layer boundaries — that's the data telling you where layers naturally touch.

---

## What already exists

The experiment is closer to runnable than it might seem:

- **Layer 1** exists: PMI co-occurrence graph from TinyStories, used in the geographdb experiments
- **Layer 2** exists: SVD embeddings of the PMI graph, 3D token positions
- **Layer 3** exists: directed transition matrix, already built and tested (showed same ~50.5 ppl ceiling as PMI — a result that itself says something about what these layers can and can't do alone)
- **Curvature tensor**: already added to `geographdb-core` as a per-token feature field

Missing: Ollivier-Ricci curvature computation on the inter-layer edges, and the inter-layer edges themselves.

The plan:
1. Build inter-layer edges: identity mapping, same token in Layer 1 → same node in Layer 2
2. Compute Ollivier-Ricci curvature: within each layer, then across inter-layer edges
3. Plot the curvature distribution — within layers vs inter-layer
4. Identify what data sits at high-curvature inter-layer points
5. Report what the geometry says, not what we expected it to say

---

## What this would mean if it works

If inter-layer curvature aligns with within-layer curvature at the same concepts without being imposed, it means the geometry of the data naturally encodes where abstraction levels interact. The routing doesn't need to be learned — it emerges from the structure.

That's a different kind of efficiency than quantization or sparse attention. Those reduce compute by approximating the matrix. This replaces the matrix with a structure that computes less because most concepts don't require multiple abstraction levels to predict.

Whether it works is an empirical question. The experiment will say.
