---
layout: post
title: "Graph-Native Reasoning Experiments: What Worked, What Didn't, and Why"
date: 2026-06-11
categories: engineering
---

Over three months I ran a series of experiments testing whether graph-based reasoning -- not transformers, not neural networks, but pure graph traversal with edge-weight reinforcement -- can solve problems that normally require deep learning. The repo is at [github.com/oldnordic/ai](https://github.com/oldnordic/ai), 62K lines of Rust, 96 standalone experiment binaries, every result logged with the failures alongside the wins.

This post is the timeline. I'm not claiming the approach replaces transformers. I'm documenting what happened when I tried.

---

## Phase 0: Spatial inference substrate (March 27)

The starting hypothesis: if tokens have positions in 3D space, and semantically related tokens are nearby, then spatial navigation through that space could substitute for attention. No O(N^2) self-attention, no massive weight matrices. Just graph traversal.

I built a spatial reasoning engine with four components:

| Component | What it does | Latency |
|-----------|-------------|---------|
| Coordinate mapper | Token to 3D position | ~1µs |
| Chunk router | Predict which spatial regions to load next | <1µs |
| Node processor | Tiny MLP (64x32) per node | 32µs |
| Learning rule | Competitive Hebbian plasticity | O(path length) |

The first validation: related code symbols (from [magellan](https://crates.io/crates/magellan)'s index) ended up **32.2% closer** in 3D space than unrelated ones. 114 symbols from 8 source files, 89 mapped to coordinates. Semantic locality was real, not theoretical.

But this was a toy. The real question was whether spatial reasoning could do useful work.

---

## Language from scratch (March 27-28)

I gave the system a blank graph and asked it to learn to assemble words. No embeddings, no pre-training. Characters are nodes, edges connect character sequences.

**CAR from nothing:** Train the graph that C→CA→CAR is a valid path. Then ask it to assemble CAR. It works. Then ask for CAT -- that works too, because the shared prefix C→CA was already learned.

This taught me something important: **compositionality comes from shared sub-paths, not from vector arithmetic**. The system doesn't "know" that CAT is related to CAR. It has a learned edge C→CA, and from there it can reach either CAT or CAR. The graph topology is the knowledge.

### What failed: HDC hypervectors

I tried Hyperdimensional Computing (HDC) for semantic guidance -- give each character a high-dimensional vector, use similarity to guide navigation. Result: **all character signatures converged to identical vectors** (similarity 1.000 for every pair). Over-bundling in the binding operation collapsed everything.

The assembly still worked -- because structural edges (C→CA→CAR) are independent of semantic guidance. This proved the two-layer architecture: a robust structural layer that works regardless of what the semantic layer does.

### What worked: Tournament selection

8 parallel solvers traverse the graph simultaneously. Each solver makes local decisions (which edge to follow). The best path wins. At 5,000 words (12,056 nodes, 12,055 synapses), success rate stays at 83.3%. Linear scaling.

The system scales because the tournament is embarrassingly parallel. Each solver only needs local graph neighborhood. No global state, no coordination overhead.

---

## Biological learning mechanisms (March 28)

I added three mechanisms inspired by neuroscience:

| Mechanism | What it does | Effect |
|-----------|-------------|--------|
| Hebbian LTP | Strengthen synapses on successful use | Path specialization |
| Synaptic decay | Weaken unused synapses by 1%/round | Homeostasis |
| Reciprocal inhibition | Suppress competing synapses when one fires | Disambiguation |

### The honesty correction

I initially labeled the inhibition "winner-take-all." It wasn't. When assembling both CAR and CAT through the shared CA node, both pathways get used and both get suppressed when the other fires. The result is reciprocal inhibition at intermediate values, not elimination. I corrected the documentation.

The scale stress test (100 → 500 → 1,000 → 5,000 words) showed constant 83.3% success rate across all scales. Bottleneck nodes grow linearly, but the tournament handles them. The hub-and-spoke topology that I was warned would fail at 1,000 nodes? **100% success rate at all scales**. The warning was wrong.

---

## Proof exploration and algebra (March 28-29)

I gave the system arithmetic and algebraic rules and asked it to discover proofs via graph traversal. The graph has mathematical facts as nodes, implications as edges.

**Rule-template generalization** was the key result. Given training on specific proofs (2+3=5, 4+1=5), the system generalized to novel inputs (7+8=15) via learned rule templates. Ablation study:

| Condition | Generalization |
|-----------|---------------|
| Full system | 1.0 (perfect) |
| No templates | 0.0 (nothing) |
| Template confidence 0.01 | 0.0 (noise) |

Templates are necessary and sufficient. Without them, the system memorizes specific paths but can't abstract.

### Template timing bug

The initial implementation applied rule templates during training, which meant training exploration weakened the template-created edges back to noise. Template confidence was also set to 0.01 -- same as bootstrap noise -- so template-backed conclusions couldn't be distinguished from random distractors. The fix: apply templates *after* all 20 training rounds, and bump template confidence to 0.05 (5x signal over noise). The checked-in `results/ablation.json` is the historical snapshot -- I marked it as such rather than quietly replacing it.

---

## Goal-directed code synthesis (March 30)

Given only I/O test cases (behavioral specs), can the system discover correct Rust functions through graph search?

The search space: 13 slot-0 ops x 13 slot-1 ops x 5 slot-2 selections = 845 programs per goal. UCB (Upper Confidence Bound) exploration with bandit-style edge weight reinforcement.

**Discovered programs:**

```
double:    n * 2              // 1 op
negate:    -n                  // 1 op
square:    n * n               // 1 op
succ:      n + 1               // 1 op
clamp_pos: n.max(0)            // 1 op
abs:       -n; a.max(n)        // 2 ops (composition!)
is_even:   n % 2; (a == 0) as i32  // 2 ops (composition!)
```

7/7 goals solved (best across 3 seeds). The hard goals -- `abs` and `is_even` -- require 2-step composition. The system discovered composition from behavioral feedback alone. No gradients, no neural networks. Pure graph search with UCB.

The honest result: per-seed average was 6.33/7 (90.5%). Seed 42 failed `square`, seed 44 failed `double`. Single-seed reliability is imperfect. Best-across-seeds is 100%.

---

## The LLM bridge experiment (March 30)

I connected the graph reasoning engine to a local model (Qwen2.5-0.5B via llama.cpp) to see if the LLM could validate candidate edges or suggest new ones.

**Oracle (validation):** 14/15 arithmetic edges validated correctly. Only 17/56 character edges -- the 0.5B model is too small for character-level reasoning.

**Reasoner (suggestion):** 2 edges suggested. A 0.5B model doesn't have enough capacity for meaningful graph reasoning.

The key engineering lesson was the chat template: Qwen2.5's BOS token (151643) is also an End-Of-Generation token. Using `AddBos::Always` caused immediate termination. Had to switch to `AddBos::Never` and manually wrap in `<|im_start|>` format.

---

## Mamba ablation (April 2)

Fixed-recurrence (RNN) vs selective-recurrence (SSM/Mamba) gating, validated via NumPy. The system uses UCB exploration to discover that optimal gating parameters (w=2.5, b=-2.5) let the state ignore noise and only absorb high-magnitude signals.

| Model | Final Error |
|-------|------------|
| RNN (fixed decay) | 19.1253 |
| Mamba (selective gating) | 0.6907 |

**96.4% improvement.** The selective gating matters. But this is a toy signal-processing task, not language modeling. The ablation confirms the mechanism works; it doesn't prove Mamba beats transformers on real data.

---

## Code intelligence with embeddings (April 12-13)

I tested whether the spatial retrieval approach works for semantic code search. 2,377 Rust symbols from the magellan codebase, embedded with Jina Embeddings v2 (768 dimensions).

**First attempt:** Use PCA to project 768D embeddings to 3D spatial coordinates, then search by spatial proximity. **Failed.** The first 3 PCA dimensions don't preserve semantic relationships. Cosine similarity on full 768D works perfectly, but spatial distance on 3D doesn't.

**Second attempt:** Louvain community detection on the similarity graph. Build a graph where symbols with cosine similarity > 0.5 are connected. Run Louvain to find communities. Map communities to spatial regions (same community = spatially close). Then spatial search finds the right community, and exact cosine similarity ranks within it.

| Query | Top Result | Precision@5 |
|-------|-----------|-------------|
| `compute_dominator_depth` | `compute_depth` (sim: 0.846) | 100% |
| `detect_cycles` | `find_cycles` (sim: 0.892) | 100% |
| `find_dead_code` | `find_dead_code` (sim: 0.931) | 100% |
| `insert_cfg_blocks` | `insert_cfg_block` (sim: 0.888) | 100% |

The lesson: **don't destroy high-dimensional information by projecting to 3D**. Use the graph (Louvain communities) to organize space, and keep full embeddings for ranking. The spatial coordinate is for navigation, the vector is for accuracy.

But the threshold calibration matters enormously. Code embeddings have mean similarity 0.37 with 90th percentile at 0.53. Thresholds below 0.55 collapse everything into one giant community. Meaningful structure (150+ communities) only emerges at threshold 0.70.

---

## Wiki-scaled graph (May 17)

The final experiment: merge wiki domain knowledge (1,999 chunks from 481 Logseq files) with the original 350-prompt activation graph. Resulting graph: 211,045 nodes, 1,055,225 spatial edges.

**0.00% repetition** across all 6 traversal modes (sequential, spatial, hybrid, conditional, verified, adaptive). Down from 8.6% on the original 27K-node graph.

At sufficient scale, the graph becomes so dense (1M spatial edges) that the walker always has somewhere new to go. The "phase boundary" between 8.6% and 0.00% repetition is the open research question: what is the minimum scale where topology alone eliminates repetition?

---

## PEMDAS discovery (May 14-15)

Can the system discover operator precedence from examples, without being told the rules?

Each operation starts with equal priority (0.5). One generic evaluator: sort operations by priority (highest first), apply in order. Coordinate descent: adjust one operation's priority, measure accuracy on all training data. Keep improvements, revert regressions.

**Learned priorities:**

```
^     : 0.730   (binds tightest -- evaluated first)
*, /  : 0.614   (multiplicative)
+, -  : 0.400   (additive -- evaluated later)
<, >  : 0.300   (comparison -- binds loosest)
```

The hierarchy emerged: `^ > */% > +-> > <==`. PEMDAS was never in the code. It was discovered from examples.

The initial version (v1) tested on 6 novel mixed-arithmetic expressions and got 100%. Later versions expanded the grammar: v3 added parentheses, exponents, unary minus, and variables (4 ops → 6 ops). v4 added comparison operators (`<`, `>`, `==`) and discovered they bind looser than arithmetic. v5 added `sin`, `cos`, `sqrt` functions, which the system discovered bind tightest of all via parse-tree recursion. The final test set is 40 expressions covering all these operators, and the system scores 100% on the lot.

This is a narrow result -- the grammar is small (9 operators + 3 functions), the training data is unambiguous, and the coordinate descent converges in ~10 rounds. It proves precedence can be *discovered* rather than hardcoded, but it doesn't scale to natural language ambiguity.

---

## What I learned

**1. Graph topology is the substrate, vectors are the ranking.** The spatial coordinates tell you *where to look*. The high-dimensional embeddings tell you *what's relevant*. Neither alone is sufficient. Together, they work.

**2. Compositionality comes from shared sub-paths.** Not from vector arithmetic. Not from attention patterns. The fact that C→CA is shared between CAT, CAR, and CARE is what makes the system compositional. The graph encodes this directly.

**3. Failures are more informative than successes.** The HDC failure taught me that the structural layer doesn't depend on semantics. The hub-and-spoke "failure" that never happened taught me to verify claims experimentally. The template timing bug taught me to check when knowledge gets applied, not just what knowledge exists.

**4. The approach has real limits.** It doesn't generate text. It doesn't understand context the way a 70B model does. What it does is retrieve, compose, and discover -- deterministically, auditably, without a GPU. For code intelligence (search, impact analysis, pattern discovery), that's enough.

**5. Honesty in research logs matters.** Every experiment in the CHANGELOG has the failures next to the successes. The "winner-take-all" correction, the data leakage fix, the HDC convergence failure. The point isn't to look good. The point is to know what actually works.

---

## By the numbers

| Metric | Value |
|--------|-------|
| Lines of Rust | 61,945 |
| Experiment binaries | 96 |
| Commits | 41 |
| Development span | March - May 2026 |
| Best precision (code search, tested queries) | 100% at 5 |
| Best scale (word assembly) | 5,000 words, 83.3% success |
| Wiki-scale graph nodes | 211,045 |
| Graph repetition rate | 0.00% |

The code is at [github.com/oldnordic/ai](https://github.com/oldnordic/ai). Everything is reproducible. The failures are in the CHANGELOG.
