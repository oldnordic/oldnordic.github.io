---
layout: post
title: "Three milestones said the reward channel was broken. It never was."
date: 2026-08-13
categories: graph-learning memoria
---

For three straight milestones, training a walker on Memoria's graph made it worse, not better. Every arm that let the contrastive reward channel touch the graph came out behind a walker that never learned anything at all. The obvious read is "the reward signal is broken, throw it out." I didn't buy that, mostly because the channel itself was simple enough that I couldn't see how it would be wrong on its own.

So instead of replacing it, I went looking for what was actually feeding it.

---

## What was really happening

The walker doesn't just walk the static graph. When it's live, it grafts new dynamic edges onto the graph as it goes — a lightweight way to let experience reshape structure without retraining anything. Grafting was never deduplicated. Every time the walker landed on the same pair of nodes again, it stacked another edge on top of the last one instead of reusing it.

On its own that sounds harmless. It isn't, because the reward channel emits one reward per *entry* in the candidate list, not one reward per *node*. A pair that got grafted five times contributes five rewards instead of one. The walker wasn't learning from the content of the reward. It was learning from how many times a pair happened to get re-grafted — which tracks graph churn, not correctness.

I checked how bad the duplication actually was: 82.9% of graft writes were duplicates of an edge that already existed. One hot pair got re-created 229 times. The reward channel was fine. It was being fed garbage at 5x volume.

---

## The fix and the numbers

One change: deduplicate the reward at the input, one reward per node per event, not one per stacked edge. Everything else in the walker stayed the same.

| arm | walker score | vs. no-learning baseline | read |
|---|---|---|---|
| baseline (no learning) | 0.095716 | — | reference |
| unfixed, learning on | 0.086011 | −0.0097 | harm reproduces |
| reward deduplicated | **0.103748** | **+0.0080** | first positive result |
| graph structurally deduplicated | 0.096386 | +0.0007 | harm removed, flat |

The pre-registered bar for "this counts as working" was +0.004. The fix cleared it at 2x.

The mechanism check is the part I trust more than the eval number itself: with the fix, positive rewards land at exactly 1.000 per update call, 0% imbalance between positive and negative magnitude. Without it: 3.48 rewards per call, magnitudes off by 4.6x, on the identical amount of graft activity in both runs. Same graph churn, wildly different reward accounting — that's about as clean an instrument check as this kind of experiment gets.

The fourth row is its own finding. Deduplicating the *graph structure* instead of the reward removed the harm too, but only got back to flat — and it turned out that's because on this substrate, grafting never added anything new in the first place. Every "new" edge the walker grafted was already sitting there as a static edge. Once duplicates were removed structurally, there was nothing left to graft. The walker with structural dedup is, byte for byte in outcome, the no-graft walker.

---

## What I don't know yet

The reward-fix arm beat the structural-fix arm by 0.0073. One run isn't enough to say that gap means anything — it goes in the notes as unreplicated, not as a result.

And this was all on one substrate: word-level bigrams. Whether the same fix holds on a subword tokenizer, or at the scale a real notes graph runs at, is the next thing to actually run, not assume. Early reads on a subword substrate say the bug generalizes — duplication was worse there, not better — but the full verdict isn't in yet.

Three milestones of "this doesn't work" turned out to be one line of code duplicating the thing being learned from. I'm glad I didn't throw the channel out.
