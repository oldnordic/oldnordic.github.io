---
layout: post
title: "The fix that should have worked made things worse"
date: 2026-08-15
categories: graph-learning memoria
---

After the reward-dedup fix from a couple days ago, one known problem was still sitting there: the same edge in the graph can get pulled into service by more than one context, and a single weight on that edge can't represent what it means in each context separately. Push the weight up for one context and you've silently pushed it up for all the others too, whether that's right for them or not.

The fix looked obvious: make the update context-aware. Instead of nudging one flat number, nudge it in a direction computed from the context that's actually active. I was confident enough in the design that I expected this to be the milestone where the graph channel finally pulled ahead cleanly.

It didn't. And the way it failed turned out to matter more than a clean pass would have.

---

## The first run lied, and the instrument caught it

Before trusting any accuracy number out of this kind of experiment, I log the polarity of every reward update — how many positive vs. negative events fired, and whether their magnitudes balance the way the math says they should on a healthy channel. That check exists specifically because a channel can look like it's learning while quietly measuring something else, which is exactly what happened three milestones ago.

First full-dose run of the context-aware update: 57% of the polarity log rows for the new update path had event counts recorded but no magnitudes — a defective logging path, not real data. A second arm, run with shuffled context as a placebo control, showed a 67% imbalance between positive and negative magnitude on a code path that should have been symmetric by construction.

Verdict on that run: void. Not "worse," not "better" — untrustworthy, thrown out without being interpreted. The accuracy numbers underneath it were pointing in a bad direction anyway, but a broken instrument doesn't get to cast that vote.

---

## The rerun, and the real number

Fixed the logging, reran. This time the instrument came back clean — polarity balanced to zero imbalance, placebo arm quiet, no non-finite values anywhere in half a billion update events.

| arm | walker score | vs. no-learning baseline | read |
|---|---|---|---|
| baseline (no learning) | 0.095716 | — | reference |
| reward-dedup only (prior fix) | 0.103748 | +0.0080 | known-good, reproduced |
| context-aware update | **0.071285** | **−0.0325** | fails badly, clean instrument |
| shuffled-context placebo | 0.092369 | −0.0114 | quiet, as expected |

The context-aware update didn't just fail to beat the existing fix. It landed below the walker that never learns anything at all. Whatever the update was doing when it fired, it was actively wrong more often than not.

Digging into why: 462,685,018 of the update's own trigger events resolved to a zero-direction no-op — the context needed to compute a direction was empty or wildcard almost every time. So the mechanism barely acted. On the times it did act, it moved the wrong way.

---

## What I don't know yet

The failed direction is closed on this substrate, pre-registered and instrument-clean, which is the outcome I actually wanted even though it isn't the one I expected going in — a clean no beats a dirty maybe every time.

The next attempt takes a different shape: instead of one shared weight nudged by context, let the edge remember which contexts it's actually served well, and bias selection toward the context that worked last time rather than trying to compute a correction in the moment. Early read at small scale: +0.0040 over the reward-dedup baseline, inside the pre-registered ±0.005 noise band — not a result yet, just a lean. Whether it holds at full scale is the open question, and this time I'm not going to guess the answer before running it.
