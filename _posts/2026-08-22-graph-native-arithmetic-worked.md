---
layout: post
title: "The same graph that couldn't write a sentence solves its own arithmetic exactly"
date: 2026-08-22
categories: graph-learning memoria
---

Last post was the negative arc: twelve mechanisms, twelve failures, trying to get a weight-free graph substrate to generate open-ended language. Same substrate, same pre-registration discipline, same independent checker — pointed at arithmetic and symbolic computation instead. This is where it stopped being a curiosity and started being an actual result.

Same rule as last time: nothing below is a vibe, every number has a spec and a control behind it, and the independent checker recorded zero false accepts across the whole arc — roughly 600,000 traces, never once certified a wrong answer.

## Growing a compute graph from demonstrations, not hand-building one

I'd already hand-built a "compute walk" graph for exact addition earlier and gotten greedy routing to 92.97% of ceiling — good, not exact, and I was honest at the time that pure greedy needed search on top, not just structure. The real question was whether the graph could grow itself from demonstrations instead of me building it by hand. It can: the grown graph came out identical to the hand-enumerated ground truth, 3,641 out of 3,643 edges, 100% accuracy at radius 3. I ran a controlled ablation withholding one region of demonstrations, and the resulting gaps landed exactly where that region predicted and nowhere else. The write path builds the compute substrate from examples with no slack in it.

## Composition generalizes to inputs it's never seen

A graph that memorizes 2-digit addition is a lookup table, not a result. So I tested composition: multi-digit addition built as chained sub-walks, from only 100 demonstrated atomic examples. On 8,100 unseen 2-digit sums: 8,055 out of 8,055 — 100% of the reachable ceiling. The 45 sums outside the ceiling aren't errors, they're an honest substrate boundary I found and reported (a landing-space limit), not something I hid. Zero false accepts or false rejects, including on carry-type integrity — the checker verifies the kind of operation, not just whether the final digit happens to match.

## Inducing the rule itself, not just applying it

Everything up to here assumed the graph already knew what carrying means. I removed that assumption: showed it 50 answers only, no worked steps, no stated rule, and it induced the exact carry table and held 100% across two independent sweeps of 8,005 pairs. My original control turned out to be mis-specified (it tested rule-independent pairs by mistake), but redone properly the comparison is unambiguous — induced-and-carry-aware hits 100%, a carry-blind control hits 1.46%. Getting every case right from answer-only supervision turned out to be a sufficient training signal on its own.

## Same mechanism, a real external dataset

Synthetic sums are a controlled sandbox. I pointed the same induction mechanism at 15,713 real examples from DeepMind's `mathematics_dataset`, multiple operators mixed together. Result: 100% operator identification, exact carry and borrow tables recovered at every dose I tried. Every case that looked ambiguous on the surface I checked with a direct collision dump, and every one turned out to be a genuine mathematical coincidence — a real 0/0 or 0×0 — not a modeling gap. This is the phase where it stopped being "works on toy data" and became "works on a benchmark other people would recognize."

## Synthesizing whole unknown programs

Pushed further: instead of inducing a known operator, synthesize an unknown function from input-output pairs alone. Five hidden functions, twelve examples each, no access to how they were generated. 5 out of 5 recovered, 100% held out. One of the recovered programs came out simpler than the actual hidden generator — genuine minimality, not overfitting to what it saw. One case that looked ambiguous turned out to be true semantic equivalence between two valid programs, and I verified that directly rather than assuming it.

## Inventing new primitives, and knowing when it can't

This is the one I'm proudest of. Give the system a fixed toolkit of base operations. When a target function can't be built from what's available, does it invent what's missing, or does it fake it? Logged exhaustion of the base library correctly triggers a constructor that invents x², x²+3, and x⁴ as new primitives, each verified 100% on held-out cases. Then I gave it a target needing x³ — genuinely outside what the current primitives can compose — and it returned no program at all, instead of guessing or forcing a bad fit. The control condition never triggers the constructor when it shouldn't. A system that expands its own toolkit only under real pressure, and reports honest failure instead of confabulating, is a much bigger deal to me than one more percentage point of accuracy anywhere else in this project.

## Pushing exactness to where the engine actually breaks

Last test: order-of-operations arithmetic, the kind of expression a naive left-to-right evaluator gets wrong constantly. Routing through structure hit 96.19% against a 55.29% naive control, with zero ordering errors across 2,100 expressions — every one of the 80 failures was the engine hitting a capacity wall (operand too large), never a wrong-order answer. I found the specific wall (a 99-iteration cap on multiplication) and broke it with decomposition: accuracy went to 99.90%, 2,098 out of 2,100, still zero wrong answers, the checker still zero false-accepts all the way through. The two remaining misses are both honestly-reported envelope limits, not silent wrongness.

## What this actually adds up to

Seven phases, one continuous chain: grow a graph from traces, compose it into multi-step arithmetic on inputs it's never seen, induce the operator from answers alone, confirm that holds at scale on a real 15,713-example benchmark, synthesize whole unknown programs from I/O pairs, invent new primitives only when actually forced to, and push precision to 99.90% with every remaining error accounted for. The same independent checker that gated all of this never once accepted a wrong answer.

Put next to the last post, I don't think the finding is "the graph is smart" or "the graph is dumb." It's a boundary, and I can point at exactly where it is: symbolic, compositional domains route through structure alone; open-ended language generation needs a trained substrate this graph doesn't have. I tested both sides with the same rigor. Only one side was expected to fail going in, and it did, exactly as specified, for reasons I can now write down instead of just wave at.
