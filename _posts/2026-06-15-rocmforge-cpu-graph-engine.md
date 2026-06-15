---
layout: post
title: "rocmforge: GeoGraph Execution Engine and Branch Selection with a 0.5B Model"
date: 2026-06-15
categories: rocmforge
---

This post covers two things built in rocmforge over the past week: a graph-based CPU execution engine with temporal rollback, and a first working result using a local 0.5B model to select between branches.

---

## The execution engine

The core problem: CPU inference traces are sequences of operations on tensors. If you want to explore multiple continuation branches from the same prefix, you need to capture the prefix state and replay it without re-executing from scratch. You also need to roll back to the prefix state after evaluating a branch.

The implementation is `CpuGraphArena` with `CpuGraph`:

- `CpuGraphArena` owns all captured tensor bytes. Handles (`F32Handle`, `U8Handle`) are stable arena offsets, not raw pointer addresses. Pointer arithmetic on arena data is safe through the handle abstraction.
- `CaptureContext` copies inputs and outputs into the arena during capture.
- `CpuGraph::execute_window(&mut arena, window)` replays a slice of the captured graph.
- `graph.regress(t)` invalidates all nodes after timestamp `t` and restores arena bindings. Rolling back to prefix state is a single call.
- `read_back` copies the final arena state to caller buffers.

Verified: `test_cpu_graph_parity` max abs error = 0.00000000. The graph replay is numerically identical to direct execution.

Benchmark (10 samples, single layer):

| Mode | Latency |
|------|---------|
| Direct imperative | ~689 µs |
| GeoGraph replay | ~709 µs |

~3% overhead from the arena indirection. The prefix capture + branch rollback pattern costs nothing at replay time beyond this baseline.

The search/rollback test: capture shared prefix → evaluate branch A → `regress(t)` → capture and evaluate branch B → both match direct execution. That test passes.

---

## Branch selection with a 0.5B model

With rollback working, the next question: can a local model pick which branch is better?

Three attempts failed before finding what works. The failures are worth documenting because the root causes are non-obvious.

**What failed:**

1. **Numeric-score-only prompts.** Asking the 0.5B instruct model to compare raw decimal scores ("Branch A: 6.2, Branch B: 5.8, which is better?") and answer with a single letter. The model either ignored the format, repeated the number, or answered randomly. The model has no useful representation for "6.2 is better than 5.8 as a branch score" — it's not a task it was trained on.

2. **Logit extraction at the wrong token position.** The label token (A or B) was not the first response token. Extracted logits were dominated by format words ("CHOICE", "Choose", digits), not the A/B decision.

3. **4-branch multi-class task.** The hidden state does not linearly separate arbitrary numeric scores at this scale. Too many classes, not enough signal.

**What works:**

- Semantic two-branch task: one branch described as moving toward the target direction, the other away. Natural language descriptions, not raw scores.
- Chat template applied before tokenization so the instruct model actually follows the prompt.
- `BranchChoiceHead`: a small linear binary classifier trained on the final hidden-state vector of the full multi-branch prompt, on top of the frozen 0.5B model.

Result on the integration test:

| | Correct / 8 |
|--|--|
| Trained choice head | 8 |
| Random baseline | 4 |

The mechanical property holds: the trained head picks the correct branch more often than random. This is on a toy task with a small held-out set. It demonstrates the mechanism works, not that it generalizes broadly.

The key constraint from this experiment: **the 0.5B model needs semantic framing.** Branch descriptions must describe what each branch does in natural language. Raw numeric annotations don't activate useful representations. This shapes everything downstream — any annotation stored in a GraphMap needs to be semantically meaningful, not just a score.

---

## What's next

The gap in the current system: real inference sessions don't yet capture a GraphMap. The branch selection mechanism works on synthetic traces. Wiring `CaptureContext` into the actual CPU inference path is what closes the loop — real forward passes produce real traces, real traces feed the branch selector, the selector's choices get stored as annotations.

After that: token-level reranking. At each decode step, take the top-N candidate tokens, run a short forward pass for each, score the resulting hidden states with a trained value head, bias the logit distribution toward higher-scoring candidates. Cost is N forward passes per generated token. Whether that cost is worth the quality improvement is an empirical question not yet measured.
