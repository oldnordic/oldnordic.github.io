---
layout: post
title: "GKH: a narrow-waist agent kernel, and the six bugs hiding inside it"
date: 2026-08-22
categories: agentic-ai gkh
---

GKH (Grounded Kernel Harness) is my agent harness for SWE-bench-style tasks: instead of giving a model a general-purpose shell and hoping, it gets a small, fixed set of verbs — `k.read`, `k.patch`, `k.search`, `k.exec`, `k.context`, `k.investigate`, `k.recall` — and every call returns a structured, budgeted response instead of raw text. The bet is that a narrow, well-instrumented interface produces more diagnosable failures than a wide one. This post is about what happened when I actually went looking for those failures instead of assuming the interface was fine.

## The architecture in one pass

On top of the verb kernel sits a `SupervisorMetaAgent`: an evolutionary loop that runs multiple generations per task (three generations, four turns each, by default), carries forward a lineage of what failed and why, and — when a generation actually resolves the task — is supposed to synthesize a reusable skill from it. Each turn goes through a driver that steers the model with GCOT (guided chain-of-thought): a live rule-interception layer that watches the model's output stream and injects correction messages (`[HARNESS RULE ...]`) when it detects a known bad pattern — claiming success without verification, deferring work, moving the goalposts.

Resolution itself is judged by the SWE-bench Verified gate: a patch only counts as resolved if every fail-to-pass test now passes, every pass-to-pass test still passes, and the patch is non-empty. No partial credit, no benefit of the doubt.

That's the design. Here's what I found wrong with the implementation, in the order I found it.

## Bug 1: the hang that had nothing to do with the model

First report from a run: it just stopped moving. My first instinct was to blame the model — infinite loop, bad prompt, something. Checked `/proc/<pid>/status` instead of guessing, and found `State: D` — uninterruptible disk sleep, not a CPU spin, not a network wait. `k.context()` was building a fresh SQLite index (`grounded_index`) for the workspace on every call, and on the machine it was running on — btrfs, copy-on-write, spinning disk — that index build hung over 20 minutes mid-write with no exception ever raised. A hang doesn't trip a try/except; the fallback path I'd written for exactly this case never got a chance to run.

Fix: `context()` is now a thin alias for `investigate()`, the AST/magellan-backed path that had been running reliably the entire session. No second unproven index-building path. If it's not adding real value over `investigate()`, it shouldn't exist as a separate implementation at all.

## Bug 2: an unbounded subprocess call

Same "it's stuck" symptom, different cause, found while auditing the same code path: `evolve_solution()` called `git diff` with no timeout. On a slow filesystem, or a genuinely enormous diff, that call can block forever with nothing upstream able to see it happening. Added `timeout=15`. Small fix, but it was sitting directly in the hot path of every single generation.

## Bug 3: token counts that were always zero

Every run had been reporting suspiciously round token totals. Traced it to `worker_result.get("tokens", 0)` — the actual usage data lives nested under `worker_result["usage"]["total_tokens"]`, so this line always silently fell back to the default. It never raised, never warned, just quietly under-reported every session. Fixed both call sites and confirmed against live llama-server telemetry that the numbers now actually match reality.

## Bug 4: skill synthesis had never once worked

This is the one that stung. The supervisor's whole evolutionary premise includes "when a task resolves, turn it into a reusable skill." A Pyright diagnostic surfaced after an unrelated edit flagged a type mismatch: `learn_skill()` requires `recipe: list[dict]`, and the call site had been passing `list[str]` the entire time. That means every prior "resolved" run in this project's history had silently failed to register a skill — the exact feedback loop that was supposed to make later generations faster never actually closed. Fixed the recipe shape to proper `{"verb": ..., "note": ...}` dicts. I didn't go looking for this one; static analysis found it, which is its own small argument for not skipping type checks on code you're confident about.

## Bug 5: the model wasn't dumb, the prompt was lying to it

A run came back with three of its four turns spent confirming `pwd`. My first read was "model capability issue." I was wrong, and only found out because I was pushed to actually check — read the raw turn content instead of the summary, and found the model repeatedly trying `cd /home/user`, a path lifted straight from the SWE-bench problem statement's own example text, which doesn't exist in the actual sandboxed workspace. It wasn't confused about the task. It was chasing a hallucinated instruction that the harness itself had handed it verbatim. Fixed by injecting the real workspace root into the generation prompt with an explicit "don't `cd`" instruction. This is the bug that made me distrust every prior "it's a model limitation" conclusion in this project until I'd actually read the logs.

## Bug 6: turns that never showed up in the log, and an echo loop hiding behind them

Adding logging to chase bug 5 revealed a second, older problem: any turn where the model produced no valid code block, or repeated a previous exact code block, was silently `continue`d with zero log output. Multiple runs had unexplained gaps in their turn history because of this — not missing data, just never printed. Once I added logging to both paths, a new pattern showed up immediately: the model was echoing the injected `[HARNESS RULE ...]` correction text back verbatim, twice, instead of acting on it. GCOT's correction mechanism assumed the model would treat an injected rule as an instruction; it was treating it as something to repeat. Added an explicit anti-echo message for exactly this case.

## What actually changed after all six

217 tests passing, same as before — none of these were caught by the existing suite, because the existing suite didn't have a scenario for "the disk hangs," "the dict key is wrong," or "the model echoes a correction." I added tests for two of the smaller fixes (novelty-threshold boundary behavior, lineage-scoped context rendering) alongside the bug fixes so they can't silently regress again.

None of these bugs were interesting individually. What's notable is that six of them were sitting in the same evolutionary loop at once, each one capable of producing a report that reads as "the model failed" when the actual cause was somewhere else entirely — a hung disk, a wrong dict key, a stale type, a hallucinated path, a missing print statement, an over-literal correction mechanism. Before trusting any "the model can't do X" conclusion out of this harness, I now check the harness first. Next post: the actual controlled experiment that came out of doing that.
