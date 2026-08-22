---
layout: post
title: "Twelve ways to make a graph write language. All twelve failed, cleanly."
date: 2026-08-22
categories: graph-learning memoria
---

I've been growing a graph substrate with no trained weights on the core routing mechanism — no attention, no matmul, no gradient descent on the walk itself. Edges come from demonstration traces, computation is a walk over the graph. Part two of this post (coming next) is about where that substrate turned out to be a genuinely capable arithmetic engine. This one is about the twelve times I tried to make the same substrate generate open-ended language, and every single one failed, for a reason I can point at and a number attached to it.

Every phase here was pre-registered before I ran it: a spec, a numeric kill bar, a control condition, written down first. An independent checker verifies every candidate answer against ground truth regardless of whether the phase is expected to pass or fail — across this whole program it has never once certified a wrong answer as correct. That discipline is why I trust these negatives instead of writing them off as bugs.

## The readout family — closed three times over

First idea: train a readout on top of the grown graph to pick the next character or word.

A wide learned readout hit 47.49% on seen text, which looked promising until I checked it against unseen text and never had. So I ran the unseen check properly: per-step continuation training landed at 0/271 unseen, and seen accuracy actually got worse in the process (down to 38.70%). Tried conditioning on prefix features next — rank-1 improved a bit (17.5% → 21.7%) but seen regressed further and unseen stayed at zero. Gave the readout more capacity with an MLP, thinking maybe it just needed more room — it got worse (13.3% unseen rank-1, down from already-bad). That's a textbook bias-variance signature, not a fixable hyperparameter. I closed the readout family there, then went back and tried once more with a training-free span-composition approach just to be sure it wasn't the training method — 0/271 again, the worst rank-1 I'd recorded. The floor isn't the readout design. It's that there's no trained substrate underneath any of them.

## Local credit converges to a bigram and stops

Tried a perceptron with local credit assignment next. It landed exactly on the bigram baseline — 9.19%, byte-for-byte identical to the untrained control. When I looked at what it actually learned, the weights were negative on length and positive on character counts. That's a model that rediscovered "shorter and more common is more likely," which a bigram table already knows for free. Nothing beyond that was reachable from local credit on this substrate.

## Five boundary-detection algorithms from the literature, one tournament

Next I implemented five different word-boundary-detection families I found in the literature — entropy-based, transition-probability-ratio, MDL lexicon, context-gated, associative — and ran them all against each other and against bigram. All five lost. Best was 8.27% against a 9.19% floor, and four of the five produced byte-identical output to each other regardless of which "theory" was driving them. What that told me: the walker's path through the graph decides the outcome before any stopping rule gets a vote. The boundary-detection angle is closed, not because I picked bad algorithms, but because the algorithm isn't where the decision is actually made.

## Growth licensing — measured the door shut before trying to open it

Before attempting to grow new composed units (multi-character pieces) into the graph, I ran a census first: how many multi-char-to-multi-char boundaries are even licensed by real training data under the rule I'd need? Zero. Across 1,367 eligible pieces and a 4,794-span inventory, greedy longest-match absorbs every frequent word whole — character-level glue is never recorded as evidence for a new unit. I never got to try growing anything, because the gate that was supposed to let growth start measured the door closed.

## Global credit, including one run where the gradient demonstrably worked

Saved the strongest attempt for last: global credit assignment on a sub-word substrate. First pass: 0/271 unseen, and the shuffled control actually beat the trained model on seen text (17.11% vs 13.85%) despite 52,702 real edge updates happening. Reshaped the reward and tried a hybrid walk for the second pass — this time the trained model clearly beat its shuffled control (15.70% vs 2.24%, 66,169 edges moved in the right direction, learning was real) — and it still landed below an untrained anchor overall, and unseen stayed at 0/113 composed, 0/271 total.

That second run is the one I keep coming back to. A working, provable gradient signal, and it still didn't close the gap. Reachability isn't choosability, and it survives an actual working gradient.

## Why I think this line is actually closed

One earlier phase makes the whole arc make sense in hindsight. Every language-adjacent result I'd gotten before this arc — going back months — had quietly been riding a trained neural hidden-state signal for context. I reran graph-native continuation with that signal removed entirely, nothing but graph structure. Result: structurally 0 out of 2,274, against a 9.19% bigram floor. Not close. The walker never once assembles two characters into a complete unseen word from structure alone.

That's the actual boundary I found: the graph is a real routing and composition substrate for domains with a checkable, finite structure — and it is not, by itself, a generative model of language. I tried five boundary mechanisms, three readout designs, two credit-assignment regimes, and one growth-licensing gate. All twelve are pre-registered, all twelve are closed, and I finally trust that they're closed for a real reason instead of a bug I haven't found yet.

Next post: the other half of this substrate — arithmetic, operator induction, program synthesis, and why the exact same discipline that killed twelve language attempts is what makes me believe the arithmetic results.
