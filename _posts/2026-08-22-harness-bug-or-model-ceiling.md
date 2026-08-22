---
layout: post
title: "Harness bug or model ceiling? Run the same model through a second harness and find out"
date: 2026-08-22
categories: agentic-ai gkh
---

Last post covered six real bugs I found and fixed in GKH, my agent harness for SWE-bench-style tasks — a hung index build, an unbounded subprocess, silent zero-token accounting, a skill-synthesis path that had never once actually fired, a hallucinated `cd` path the harness itself handed the model, and a correction message the model was echoing back instead of acting on. After fixing all six, GKH still scored 0/2 on two SWE-bench Verified tasks (`sympy__sympy-13031`, `sympy__sympy-12419`). The obvious next question: is that 0/2 a seventh bug I haven't found yet, or is it the actual ceiling of the model on these two tasks?

You can't answer that question by staring at one harness harder. You answer it by running the identical model through a *different* harness against the identical tasks. If the second harness also fails the same way, the harness stops being the prime suspect.

## The setup

Same two tasks, same model (Qwen3.6-35B-A3B, running locally via llama.cpp), two different harnesses:

- **GKH**: the evolutionary supervisor from the last post, turn-budgeted across three generations of four turns each.
- **OpenCode**: a completely independent, general-purpose coding agent CLI, given a single flat 900-second timeout per task and no other constraints — a more generous, less structured budget than GKH's.

Both harnesses work from an isolated git worktree per task and get evaluated by the exact same strict SWE-bench gate: fail-to-pass tests must now pass, pass-to-pass tests must still pass, patch must be non-empty.

## The result

OpenCode, same model, same two tasks: **0/2**, both runs hitting the full 900-second timeout, both producing a completely empty patch.

```
sympy__sympy-13031: TIMEOUT, patch_length: 0, resolved: false
sympy__sympy-12419: TIMEOUT, patch_length: 0, resolved: false
```

Same failure mode as GKH, on a harness with a different architecture, a different prompting approach, and a more generous timeout. That's real evidence the two tasks are genuinely hard for this model, not evidence of a GKH-specific defect — which is the answer I was actually looking for, and the reason I didn't stop at "well, GKH went 0/2, ship it."

## Trying a second model, and finding an instrumentation gap instead of an answer

To push further, I downloaded a second local model — Nemotron-3-Nano-30B-A3B, a newer MoE architecture (128 routed experts, hybrid Mamba-2/Transformer), running with CPU expert-offloading so it fits alongside everything else on one GPU — and ran it through OpenCode against the same two tasks.

Also 0/2, also both TIMEOUT at the full 900 seconds. But this run surfaced something I don't have a clean answer for yet: both attempts reported **zero total tokens used** in the usage field, despite burning the entire 900-second budget. That's not what I'd expect from a model that's actually reasoning and failing to converge — it looks more like a broken usage-reporting path for this particular model/provider combination than a model that produced 900 seconds of silence. I'm not treating that as a second capability data point until I've confirmed whether tokens were actually generated and just not counted, versus the request genuinely stalling somewhere before generation started.

## Where this actually leaves things

The controlled part of this experiment is solid: same model, two independently-built harnesses, same two tasks, same failure mode, empty patches both times. After finding and fixing six real bugs that could each have produced a false "model failed" reading, the result held steady once the harness was out of the way. That's a real, if narrow, finding — these two SWE-bench Verified instances are hard enough that harness architecture doesn't move the needle for this model.

What it doesn't tell me yet is anything solid about the second model, because the zero-token anomaly means I don't actually know what happened inside that 900-second window. The honest move is to say so rather than fold a broken counter into a "the new model also can't do it" narrative. Before I trust a number, I check that the number is measuring what it claims to measure — and this one currently isn't. Next step is fixing the usage-reporting path for that provider before drawing any conclusion about that model at all.
