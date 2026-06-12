---
layout: post
title: "MiMo Code Dreams Every 7 Days -- and the Architecture That Makes It Work"
date: 2026-06-12
categories: engineering
---

On June 10, Xiaomi released MiMo Code, an MIT-licensed fork of OpenCode with a memory architecture that's genuinely different from anything else shipping in a terminal coding agent. Chew Loong Nian published a detailed teardown on his blog; this post is my analysis of what matters, what doesn't, and where the ideas intersect with work I've been doing independently.

## The claim that matters

1,213 double-blind A/B tests. 576 developers. 474 private repositories. MiMo Code beat Claude Code past 200 steps in 65% of tasks. Under 200 steps, it was a coin flip.

That 200-step threshold is the important number. Under 200 steps, the model is doing most of the work -- the harness is just a loop. Past 200 steps, the harness *is* the product, because the model's context window has wrapped at least once and coherence depends entirely on how the harness manages state. Xiaomi's own benchmarks admit this: on SWE-Bench Pro (one-shot issue solving, no long-horizon memory needed), MiMo Code scores 62% vs Claude Code's ~57%. That gap is real but modest. The 65% number at 200+ steps is where the architectural difference shows up.

Three honest caveats before the table. First: every number is Xiaomi-reported, no independent replication exists yet, and the release is days old. Second: the offline benchmarks measure one-shot solving, precisely the regime where memory machinery does nothing -- Xiaomi admits this openly. Third: the A/B methodology is described carefully (anonymous parallel agents, developer scoring, trajectory triangulation) but the referee is also the home team.

## The architecture: three layers, four mechanisms

Xiaomi organizes the system into computation, memory, and evolution. Four mechanisms are worth understanding in detail.

### 1. Checkpoint writer: out-of-loop memory extraction

This is the most important idea in the release. The main agent is not allowed to maintain its own memory.

Xiaomi found that asking a model deep in a debugging spiral to also maintain structured notes makes it worse at both jobs. So extraction is moved to a separate subagent -- the checkpoint writer -- with its own attention and token budget. It runs at roughly 20%, 45%, and 70% of context utilization. It reads the conversation so far and writes an 11-field structured state file:

| Field | Purpose |
|-------|---------|
| Current intent | What the agent is trying to do right now |
| Next action | Planned tool call or edit |
| Working constraints | Discovered limitations (API quirks, build flags) |
| Task tree | Parent/subtask decomposition |
| Current work | Active file, region, approach |
| Involved files | All files touched in this session |
| Cross-task discoveries | Facts that may matter for other tasks |
| Errors and fixes | What broke and how it was resolved |
| Runtime state | Environment, language, framework versions |
| Design decisions | Tradeoffs chosen and why |
| Notes | Free-form catch-all |

Why checkpoint at 20% and 45%, not at 95%? Two reasons. First, "lost in the middle": extraction quality degrades when the window fills up. Compressing at 95% utilization means compressing with a degraded model. Second, extraction itself needs headroom to think. At 30% there's room; at 95% there isn't.

When the window hits its real limit, the runtime performs a *rebuild*: it kills the physical window, opens a fresh one, and reconstructs context from the persisted files -- capped at roughly 65K tokens, layered in a deliberate order (task list first, then session checkpoint, then verbatim slices of recent user messages). The model never knows the conversation ended. Xiaomi calls each checkpoint-then-rebuild span a *cycle*, and a logical session is an unbounded chain of cycles.

This is architecturally identical to what I've been building with atheneum's dream pipeline, but with a critical difference: MiMo Code's memory is file-based (Markdown), while atheneum's is graph-based (SQLite with HNSW). The tradeoff is auditability vs queryability. Markdown is trivially readable and editable by humans. Graph queries are dramatically faster for "what do we know about X" across hundreds of sessions. Xiaomi chose human auditability. I chose machine queryability. Both choices are defensible; neither is obviously better.

### 2. Goal verifier: the agent that won't let itself quit

The most common long-horizon failure isn't a wrong edit -- it's the agent declaring victory at step 140 of a 220-step task. MiMo Code's Goal mechanism lets you define a natural-language stopping condition. Every time the agent tries to terminate, an independent model call reviews the full history and checks whether the condition is truly satisfied. If not, the specific gap feeds back and the agent keeps working.

Xiaomi reports two honest failure modes: false blocking (verifier refuses genuinely finished work, usually due to flaky test environments) is more common than false passing, and the infinite-loop probability is under 0.5% with an automatic exit limit.

This is a verification layer. My stack handles this differently -- the grounded-coding-verification skill runs `cargo check`, `cargo test`, and `cargo clippy` after every edit and refuses to mark work complete until all three pass. The verification is mechanical (does it compile, do tests pass) rather than semantic (does the intent match the goal). Xiaomi's approach handles goals like "the migration is committed with a descriptive message" that my approach can't verify mechanically. The tradeoff is cost: every Goal check is an additional model call.

### 3. Max Mode: five brains, one judge

At each turn, Max Mode samples N=5 candidate responses in parallel at temperature 1. Each candidate reasons and plans tool calls -- but nothing executes. Then the same model, at low temperature, judges all five plans and picks the most robust one to execute.

The logic: if all five converge on the same plan, that's a confidence signal. If they diverge, a judge picks the sturdiest option. Xiaomi reports 10-20% lift on SWE-Bench Pro at 4-5x token cost. It's off by default, which is correct given the bill.

This is best-of-N sampling with a twist (the judge sees plans, not outputs). It's expensive and the gains are real but marginal for the cost. Interesting for high-stakes single decisions; impractical for long-horizon loops where you'd burn 5x tokens on every step.

### 4. Dream and Distill: scheduled maintenance

Every 7 days, Dream runs: an independent agent merges duplicate memories, deletes references to files that no longer exist, and compresses everything into a tighter representation. Every 30 days, Distill runs with a different mandate: it ignores knowledge and hunts for *process* -- recurring work patterns it can solidify into reusable skills and SOP documents.

The biological analogy is explicit and not accidental. Memory consolidation during sleep is one of the best-understood mechanisms in neuroscience. Xiaomi's implementation is a straightforward application of the same principle to machine memory.

This maps directly to what I've been building. Atheneum's `dream` command does memory consolidation (merge duplicates, remove stale entries, compress verbose content). The `wiki-dream` command does wiki page maintenance. And the code dream prototype I built -- scanning 24 magellan databases across 14 projects to find optimization patterns -- is a form of cross-session knowledge extraction that serves the same purpose as Distill: finding patterns that persist across sessions and crystallizing them.

The key difference is scheduling. Xiaomi runs Dream every 7 days on a timer. My dream pipeline runs on demand when invoked. For a single developer on a single machine, on-demand is fine. For a team with 474 repositories, scheduled maintenance is the only thing that actually happens.

## The memory stack

Xiaomi has four layers:

| Layer | Format | Lifecycle | Contents |
|-------|--------|-----------|----------|
| Session | checkpoint.md | Current session | 11-field working state |
| Project | MEMORY.md | Persistent | Architecture decisions, user rules, verified facts |
| Global | Cross-project | Cross-project | User-level preferences |
| History | SQLite | Forever | Raw text of every message and tool call |

The main agent gets read-only access to all structured files except `notes.md`, a free-form scratchpad it can append to. At each checkpoint, the writer reads `notes.md`, routes content into proper structured fields, and wipes it. Single writer per file, enforced at code level.

This is clean engineering. The single-writer constraint prevents the corruption that happens when an in-loop agent overwrites structured state while in a degraded state (which is exactly the failure mode Xiaomi designed around). The fallback to SQLite full-text search when structured memory doesn't have the answer is pragmatic.

My atheneum stack is architecturally similar but different in implementation: SQLite with HNSW for semantic search instead of FTS, graph entities with typed edges instead of flat Markdown, and a `memory_entries` table for key-value facts alongside `graph_entities` for structured knowledge. The query patterns are different but the lifecycle is the same: write during work, consolidate on schedule, compress over time.

## The funnel critique and why it doesn't matter

The HN thread's loudest complaint: MiMo Code defaults to Xiaomi's free MiMo Auto tier, phones home to `tracking.miui.com` with telemetry on by default, and is functionally a funnel for Xiaomi's model API. The author of the teardown tested it and confirmed the telemetry behavior.

The critique is true and doesn't matter. Fork-and-optimize is how open source works. KHTML became WebKit became Blink. Every cloud provider's CLI defaults to their own endpoint. The MIT license means the telemetry can be forked out -- and the author provides the exact environment variable to disable it:

```
export MIMOCODE_ENABLE_ANALYSIS=false
```

What matters is whether the architecture is novel. It is. The checkpoint writer, the scheduled Dream cycle, and the single-writer memory constraint are ideas that haven't shipped together in any other harness. Publishing them under MIT with detailed engineering data -- including honest failure modes -- is a contribution regardless of the commercial intent behind the release.

## What I'm taking from this

Three ideas worth stealing:

1. **Scheduled Dream cycle.** My dream pipeline runs on demand. It should run on a schedule. The 7-day interval Xiaomi chose is probably right for project memory; the 30-day Distill interval for process extraction is also reasonable. I'll set up a cron job.

2. **Checkpoint at 20% and 45%, not 95%.** I don't have a checkpoint writer in my agent loop, but the principle applies to any context management: compress early while the model is still coherent. This is the kind of counterintuitive insight that only comes from running thousands of long-horizon sessions.

3. **Single-writer per memory file.** My atheneum writes are already single-writer (the agent stores via CLI, not directly), but the explicit constraint that the working agent *never* writes to structured memory is a stronger version of what I'm doing. Worth considering for the Hermes plugin.

Three things I'm not taking:

1. **Max Mode.** 5x token cost for 10-20% gain doesn't pay for itself in a long-horizon loop. Best-of-N sampling is useful for single high-stakes decisions, not for every step of a 200-step task.

2. **Markdown memory format.** Auditability is nice, but I have 1,112 entities in atheneum. Full-text search over Markdown at that scale doesn't perform. Graph queries with HNSW do.

3. **Goal verifier.** The model-call-per-termination-check is expensive and the failure mode (false blocking on flaky tests) is exactly the kind of nondeterminism I try to avoid. Mechanical verification (compiles, tests pass, lint clean) is more reliable even if it can't verify semantic intent.

## The 200-step lesson

The real takeaway from Xiaomi's data isn't about MiMo Code vs Claude Code. It's that **the 200-step threshold is where harness architecture becomes the bottleneck**. Under 200 steps, any reasonable harness with a good model performs similarly. Past 200 steps, the model's context has wrapped and coherence depends entirely on state management.

This matches my experience. The code dream prototype scans 7,982 symbols across 14 projects and runs 8 detectors in ~5 seconds. But the findings are only useful if they persist across sessions and get better over time. That persistence -- atheneum's graph, the dream pipeline's scheduled maintenance, the cross-project optimization transfers -- is the part that compounds. Session 1 finds 85 patterns. Session 10, after Dream has run a few times, should find fewer duplicates and more genuine insights. Session 100 should be qualitatively different from session 1.

Xiaomi's 65% number at 200+ steps is the first public evidence I've seen that this compounding effect is real and measurable. It's vendor-reported and it needs independent replication. But the direction is right, and the architecture they describe to achieve it is sound.

The agent that dreams is a gimmick name for a serious idea: maintenance of machine memory shouldn't be the job of the agent doing the work. Biology figured that out a few hundred million years ago.
