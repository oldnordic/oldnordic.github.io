---
layout: post
title: "Transformer X-Ray, Part II: The BOS Bottleneck"
date: 2026-06-16
categories: rocmforge mechanistic-interpretability
---

The first [x-ray post](https://oldnordic.github.io/rocmforge/mechanistic-interpretability/2026/06/16/transformer-xray.html) looked at where the prediction position sends attention mass. That was useful, but it flattened the forward pass into one aggregate graph.

The more interesting question is when the routing pattern changes.

I ran a small sweep of 16 prompt pairs and compared layerwise attention divergence between an intended continuation context and a contradictory one. Three examples stood out:

- `dna_acid`
- `shakespeare_romeo`
- `wwii_end`

They produce three different depth profiles. Together they suggest a consistent structure: the forward pass compresses into a BOS-dominated bottleneck around layer 16 of 24, then reopens near the output.

This post is about that shape.

---

## What was measured

For each prompt pair, I traced the full forward pass and computed layerwise Jensen-Shannon divergence between the attention-flow distributions under:

- an intended continuation context
- a contradictory continuation context

Separately, I aggregated the prediction-position attention by layer and measured how much of that mass went to position 0 (BOS).

One caveat up front: in this sweep, "correct" means the prompt was continued with the intended context, not necessarily that the model produced the expected answer token. Two of the three examples below do **not** produce the expected final token even in the intended context. They are still useful because the claim here is about routing depth, not benchmark accuracy.

The relevant trace files are:

- `/tmp/dna_correct.jsonl`
- `/tmp/shakespeare_correct.jsonl`
- `/tmp/wwii_correct.jsonl`
- `/tmp/kappa_results.jsonl`

---

## Shared structure

Across all three traces:

- `L0-L2`: local/contextual routing dominates
- `L3`: BOS turns on abruptly
- `L16`: BOS reaches maximum compression
- `L17-L23`: BOS releases and semantic positions reopen

The normalized BOS share at the bottleneck layer:

| Example | `L16` BOS share |
|---------|-----------------|
| `dna_acid` | `0.933` |
| `shakespeare_romeo` | `0.915` |
| `wwii_end` | `0.889` |

This is the strongest invariant in the data so far. The bottleneck is not vaguely "somewhere in the middle." On these traces it sits around two-thirds depth: layer 16 of 24.

---

## Case 1: `dna_acid`

Prompt family: `"DNA stands for deoxyribonucleic ..."`

This is the clean memorized-phrase case.

Layerwise divergence:

- average JS divergence: `0.0553`
- `L2` peak: `0.1803`

That early peak matters. The contradiction shows up before the BOS sink fully activates.

Prediction-position BOS share:

| Layer | BOS share |
|-------|-----------|
| `L0` | `0.045` |
| `L1` | `0.049` |
| `L2` | `0.034` |
| `L3` | `0.510` |
| `L16` | `0.933` |
| `L23` | `0.536` |

Interpretation:

- early layers are reading a memorized lexical pattern
- BOS compression takes over at `L3`
- by `L16`, the trace is almost fully collapsed into BOS
- by `L23`, BOS is still dominant, but semantic positions re-emerge strongly

This is an early-detection contradiction.

---

## Case 2: `shakespeare_romeo`

Prompt family: `"Romeo and Juliet was written by ..."`

This is the low-confidence late-separation case.

Layerwise divergence:

- average JS divergence: `0.0503`
- `L23` peak: `0.1506`

Prediction-position BOS share:

| Layer | BOS share |
|-------|-----------|
| `L0` | `0.037` |
| `L1` | `0.146` |
| `L2` | `0.091` |
| `L3` | `0.734` |
| `L16` | `0.915` |
| `L23` | `0.416` |

Confidence on the intended-context trace is low: `0.2678`.

That shows up in the late layers. BOS weakens, but the trace does not collapse into one clear semantic target. The upper layers remain distributed.

This is not the same pattern as `dna_acid`. The contradiction is not caught early. The middle stack is comparatively stable. The separation comes late, and the final routing remains diffuse.

---

## Case 3: `wwii_end`

Prompt family: `"World War II ended in ..."`

This is the strongest late-release case.

Layerwise divergence:

- average JS divergence: `0.0402`
- `L23` peak: `0.1308`

Prediction-position BOS share:

| Layer | BOS share |
|-------|-----------|
| `L0` | `0.049` |
| `L1` | `0.121` |
| `L2` | `0.049` |
| `L3` | `0.677` |
| `L16` | `0.889` |
| `L23` | `0.101` |

The important detail here is not that BOS is literally absent in the early layers. It is not. The important detail is that BOS is negligible relative to the semantic positions before `L3`, then almost disappears again by `L23`.

At `L23`, the top positions are no longer BOS-dominated:

- position `2`: `4.808`
- position `4`: `4.333`
- position `5`: `2.070`
- BOS: `1.411`

So the late layers are reading the slot directly. The model stops using BOS as the dominant routing anchor and turns back to the content positions that matter for the year completion.

---

## The shape

The hourglass is real, but it is not symmetric.

The observed structure on these traces is:

1. **Bottom opens early**: `L0-L2` are local and content-heavy
2. **Compression begins abruptly**: BOS turns on at `L3`
3. **Neck sits late**: maximum compression is around `L16`
4. **Top reopens near output**: `L17-L23` release BOS and return mass to semantic positions

That is more precise than saying "attention becomes abstract in the middle."

More concretely:

- early layers detect lexical or contextual pattern
- middle layers compress routing into a stable transport regime
- late layers reopen that routing to make the final commitment

The three examples differ in where contradiction becomes visible:

- `dna_acid`: early lexical contradiction
- `shakespeare_romeo`: late, low-confidence separation
- `wwii_end`: late semantic slot reading

---

## What this does and does not show

What it shows:

- the forward pass has a measurable depth profile
- BOS can act as a real routing bottleneck
- different prompt types diverge at different depths

What it does **not** yet show:

- that this bottleneck is universal
- that `L16` is stable across models
- that the early/late split cleanly maps to "memorized" vs "compositional" at scale

The current sweep is `N=16`. That is enough to preserve the pattern, not enough to treat it as settled.

The next useful run is larger and simpler:

1. `50-100` matched prompt pairs
2. first divergence layer
3. peak divergence layer
4. area under the divergence curve
5. split by prompt type

If the layer-16 bottleneck and the early-vs-late split survive that, this stops being a visual anecdote and starts becoming a routing diagnostic.

---

## Files

The x-ray tracer is in [rocmforge](https://github.com/oldnordic/rocmforge). The visualization work is in [geographdb-core](https://github.com/oldnordic/geographdb-core).

Relevant local artifacts from this run:

- `/tmp/kappa_results.jsonl`
- `/tmp/dna_correct.jsonl`
- `/tmp/shakespeare_correct.jsonl`
- `/tmp/wwii_correct.jsonl`

The first post in this series is here:

- [X-raying a Transformer Forward Pass](https://oldnordic.github.io/rocmforge/mechanistic-interpretability/2026/06/16/transformer-xray.html)
