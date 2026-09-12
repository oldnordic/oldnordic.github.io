---
layout: post
title: "Nothing was delivered: three days of verified motion, one morning of actual speed"
date: 2026-09-12
categories: agentic-ai llama-rs gkh
---

I spent most of this week watching an agent fleet work. It reviewed honestly, it rejected fakes, it cited its sources, and after three days the scoreboard read: prefill still slow, correctness still open, nothing you could call a deliverable. The agents weren't lying — every claim had a receipt. They were just producing verified motion instead of results, and the difference only shows when you force the question "what can I use that I couldn't use yesterday?"

This post is what happened after I started forcing it. The short version: a four-arm experiment on reasoning budgets, one dispatch-trace that turned a vague 8x slowdown into five named gaps, a nine-minute fix that nearly doubled-and-a-half prefill, and a set of honesty exhibits I didn't expect — including a flag everyone was reasoning about that turns out to do nothing at all.

## The vocabulary I didn't have: cost per accepted task

The framing that finally organized the mess came from Adnan Masood's work on CPAT — cost per accepted task, with failed attempts, retries, and human rescue counted in the numerator. By that measure, my llama-rs lane wasn't a research program, it was a cost center with a zero denominator. Token counts, review phases, experiments dispatched: all spending metrics. The only number that matters is accepted outcomes, and the lane's acceptance rate was zero.

That sounds obvious written down. It was not obvious while the receipts were flowing in every thirty minutes, each one internally consistent.

## Four arms, one model, one task

The cleanest experiment of the week was accidental. I have a harness (GKH) where a local model writes tool-call code against a typed verb API. I pointed it at a bugfix task — fix a 19-line resolver, insert a regression test into a 2,117-line test file without rewriting it — and ran the same task four ways on the same Qwen3.6-35B-A3B (llama.cpp, RX 7900 XT, 128k context, Q8 KV, MoE experts on CPU):

| arm | config | verdict |
|---|---|---|
| 1 | thinking ON, no budget | TRUNCATION_LOOP, 4 turns, 0 file edits, ~13 min |
| 2 | thinking OFF | QA PASS, 18 turns, ~85 s |
| 3 | thinking ON + `--reasoning-budget 1024` | stalled_repetition, 8 turns, wrong patch |
| 4 | thinking ON + budget + JIT verb teaching | DONE, audit PASS, skill captured |

Arm 1 is the failure I want to describe precisely, because it's not a "loop" in any cycle-detector sense. The model reasoned 2048, then 4096, then 8192 output tokens per turn, and never once finished a code block — the thinking physically crowded out the action. With reasoning hidden, that's indistinguishable from a hang. It turns out this is a known Qwen3.x serving failure: a user-collected sample of ~700 calls on a related model measured ~17% of responses ending with an empty answer after thousands of reasoning tokens. Not my bad luck; a serving-path property.

Arm 3 is the subtle one. llama.cpp ships `--reasoning-budget N` and `--reasoning-budget-message` — a hard cap on thinking plus a message injected when the cap hits ("summarize in 3 lines, emit one verb call"). That eliminated the spiral completely: every turn now ended in an action. And the model then patched the bug *wrong* and never added the test — it emitted `parts[-2]` where the correct fix was the first directory component, producing `'src'` instead of `'proj'`, verified by me after. A brake working perfectly while the user gets nothing.

Arm 4 is what actually steered. The harness's sandbox refusals started carrying the remedy ("you cannot define functions here — to edit a file: `k.patch(path, target, replacement)`; to insert: `k.insert_at(...)`"), the scaffold gained one 216-character rule with a single legal-turn example, and a `k.help(verb)` verb made detail pull-based. No prompt bloat — the teaching arrives at the moment of failure, nowhere else. Same model, same task, same thinking-on budget: done in 12 turns, audit pass, and the episode's winning recipe got captured as a reusable skill.

The conclusion I keep reusing: changing what the model sees beats punishing what it does. Every brake I built eventually produced the dump that taught me the steering.

## The flag that did nothing

While bisecting why llama-rs decode suddenly matched llama.cpp token-for-token, I found the whole coordination trail had been reasoning about `LLAMA_RS_HIP_MOE_GATE_UP_OPT` — an environment variable echoed in the bench output, cited in reviews, treated as a live toggle. It is consumed nowhere. Not in src, not in tests, nowhere. The actual behavioral gate is `LLAMA_RS_MOE_DOWN_ACCUM`.

One `grep` would have ended that investigation at hour zero. Verify the instrument before building the theory on it — a day of careful reasoning about a dead flag is the most expensive sentence in this post.

## Eight seconds of missing work, named

The llama-rs prefill had been flat at 101–111 tok/s across five consecutive benchmark runs (llama.cpp: 837). Instead of another sweep, I had a worker produce a source-only dispatch trace: op by op, llama.cpp vs llama-rs, which kernel, which batch shape, which host syncs, every claim with a `file:line` citation. The 8.27x gap decomposed into five named causes. The big one: the MoE down projection ran 512×8 = 4,096 independent matrix-vector columns per layer, each picking its expert separately — a logical upper bound of ~112 GiB of independently-addressed weight bytes per prefill, versus llama.cpp's expert-binned MMQ where one weight tile is reused across all tokens routed to that expert. Second: the prefill router was a CPU island — 20,480 CPU argsorts and at least 40 GPU drain synchronizations per request, because a past incident (a strided-view bug that once produced a wrong first token) had banished multi-row routing off the GPU.

The fix for gap one already existed in the repo — a grouped-WMMA branch that the ordered prefill route simply never selected. Routing to it was a nine-minute dispatch with a fail-first test. The acceptance run through the fail-closed benchmark gate:

```
qwen3.6-35b-a3b  pp512  rust  101.26 -> 239.0 tok/s  (llama.cpp: 847.6)
ornith-35b       pp512  rust  117.4  -> 193.6 tok/s  (llama.cpp: 791.2)
```

Deterministic warmup/measured streams, receipts for everything. Not parity — llama.cpp is still ~3.5x ahead — but the first movement in five runs, and it came from naming the mechanism, not from another probe.

## The change that failed honestly

The second fix, for the router CPU island, passed its CPU contract test and then failed in exactly the way the gate exists to catch. Its multi-row GPU router claim made warmup and measured token streams diverge in prefill — the same failure class as the historical incident that got routing banished to CPU in the first place. The CPU test validated the claim logic against the argsort oracle; the bug lived in the device consumption path, where the CPU test can't see.

So it's quarantined: the code stays, behind an opt-out env flag, with the known issue written into the changelog. No tolerance edits, no "works on my run." It's now the next named work item — the 40 drain syncs are still on the table, and re-claiming them correctly is worth real speed.

One more exhibit from the same test suite: a pre-existing audit test prints, in its own failure output, that llama-rs's flash-attention prefill kernel runs at 9.86 ms/call against llama.cpp's 0.32 ms tile kernel — a 30.8x gap, with the arithmetic shown (4.3 GB unshared KV traffic vs 134 MB with llama.cpp's 32x shared-memory reuse). That's the biggest single number in the repo right now, and it was sitting in a failing test's stdout the whole time.

## The statistical sin, self-reported

My continual-learning experiments (can a tiny 2-bit recurrent model learn task B without destroying task A?) had a report on disk claiming 0% forgetting. Reading it as a reviewer instead of as its author: Task A was only ever half-learned (one verb of two), and Task B's eval was flat before and after training — B was never acquired either. Zero forgetting, trivially. You cannot measure the erosion of an asset that was never built.

The correction is a pre-registered re-run with frozen bars: A must reach 0.9 before B starts (or the run reports the capacity ceiling and stops), B must improve by a declared delta (or "B not learned" is the verdict), and only then does forgetting get computed. If the answer is a capacity negative at 5M parameters, that negative is the deliverable — it sets the scale for the next experiment. The machinery to fit data was already proven separately; what's being tested is the claim.

## What I'm taking from the week

The fleet didn't get smarter when I gave it more freedom or more tokens. It got useful when I started asking, per lane, the CPAT question — what did you deliver that a designated acceptor approved, and what did the failures cost? The harness matters more than the model on every axis I measured this week: a reasoning budget flag, a 216-character scaffold rule, a dispatch trace with line citations, a gate that refuses to claim success it can't re-execute. All boring machinery. All of it moved numbers; none of it required a new model.

The full receipts — benchmark JSON, test logs, the dispatch trace with citations, the pre-registered experiment spec — are in the respective repos. The numbers in this post are the numbers from the receipts; where I couldn't verify something, I've said so rather than fold it into the story.
