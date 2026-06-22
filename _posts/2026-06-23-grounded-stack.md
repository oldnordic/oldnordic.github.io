---
layout: post
title: "The Stack I Built Because Nothing Else Did This"
date: 2026-06-23
categories: engineering
---

I use AI coding assistants every day. Multiple sessions, multiple agents, sometimes parallel subagents working different parts of the same problem. After a few months of this, two things became obvious.

First: every session starts from zero. The agent that traced a subtle WAL contention bug yesterday has no memory of it today. You explain the same context again. It rediscovers the same facts. It re-reads the same files. Token budget gone, nothing learned.

Second: when something goes wrong -- a file overwritten, a wrong refactor applied, a decision made that broke something three layers down -- there is no audit trail. The answer to "what did the agent do and why?" is "check the transcript" which means loading thousands of tokens into context and reading it yourself.

I looked for tooling that solved either of these problems at a price and form I could actually use. I couldn't find it. So I built what I could.

This is the current state of that work: what exists, why it was built in this order, and where it's going.

---

## The three layers

The stack has three distinct layers, built in dependency order.

### Layer 1: Code intelligence

Before agents can make auditable decisions about code, they need a precise model of what the code is. Not a fuzzy embedding. Not "here are 5 relevant files." A structured record: which symbols exist, where they're defined, who calls whom, how control flows, what gets affected when something changes.

**[magellan](https://crates.io/crates/magellan)** is the indexer. It parses source trees with tree-sitter across 9 languages (Rust, C, C++, Java, Python, JavaScript, TypeScript, Go, CUDA), extracts symbol definitions and call-graph edges, and stores everything in a per-project SQLite database. The database is a file. You copy it, `scp` it, inspect it with any SQLite client. No server required.

From the index, other tools do their work:

- **[llmgrep](https://crates.io/crates/llmgrep)** -- semantic symbol search across a magellan database. "What handles authentication errors?" finds the answer in milliseconds using FTS5, not by scanning files.
- **[mirage-analyzer](https://crates.io/crates/mirage-analyzer)** -- control-flow graph analysis. Cyclomatic complexity, dominance trees, reachability paths, hotspot detection. CFG accuracy varies by language: C and C++ get CFG extracted from LLVM IR; Java gets CFG from bytecode. Both are compiler-verified. Everything else -- Rust, Python, Go, JavaScript -- is AST-based. Structurally useful, not semantically precise. That's the honest distinction: two languages get the real thing, the rest get a best-effort approximation.
- **[splice](https://crates.io/crates/splice)** -- span-safe refactoring. Edit operations that know about byte offsets and symbol boundaries, so you don't accidentally break adjacent code.

Before touching anything in a codebase, the workflow is: find the symbol, trace its references, check its blast zone (what else gets affected), query its call graph. This takes seconds with these tools. It took 10-20 minutes of manual file reading before.

The inspiration here is not original. Linux kernel developers have navigated millions of lines of C for decades with `cscope` (cross-reference browsing) and `ctags` (symbol indexing) -- two tools that do conceptually the same thing: build a queryable index of the codebase so you don't have to hold it all in your head. Experienced systems engineers often have the equivalent in personal scripts built up over years. What magellan and llmgrep attempt is the same discipline applied to LLM-assisted coding: give the agent a structured index it can query instead of making it read raw files. The concept comes from watching how serious engineers actually navigate large codebases. The implementation is adapted for a different kind of reader.

### Layer 2: Storage

**[sqlitegraph](https://crates.io/crates/sqlitegraph)** is a graph database on SQLite. It started as the persistence layer for magellan, then grew to implement 35+ graph algorithms (BFS, DFS, PageRank, SCC, Louvain), HNSW vector search, MVCC version chains, and FTS5 full-text indexing. The 3.3 release added temporal topology via persistent homology -- you can now track how a codebase's connectivity structure changes across indexed versions.

**[geographdb-core](https://crates.io/crates/geographdb-core)** is a separate geometric graph database implementing Wasserstein distance, Ricci curvature, and spatial graph algorithms. The research direction here is understanding codebase structure through geometry -- treating the call graph as a metric space and asking what its curvature tells you about coupling and cohesion. This is ongoing work.

Both are SQLite-backed, both are GPL-3.0, both run entirely offline on consumer hardware.

### Layer 3: Coordination and memory

Code intelligence tells you what the code is. The coordination layer tells you what the agents did, what they decided, and why.

**[atheneum](https://crates.io/crates/atheneum)** is a knowledge graph that persists across sessions. Every agent session writes tool calls, discoveries, decisions, and reasoning traces into a structured SQLite graph. Between sessions, a `session-digest` command generates a token-budgeted bootstrap packet -- the last N sessions' decisions, modified files, and open tasks -- so a new session grounds itself in prior context without reloading transcripts into the context window.

The `thread` command navigates decision chains. Given a query like "version bump decision", it finds the matching discovery node and walks the `caused_by`/`led_to` edge chain outward, bounded to a token budget. Decisions are linked to the tool calls that informed them, which are linked to the sessions that produced them, which are linked to the files that were modified. The graph is the index; retrieval is SQL.

Atheneum also has a built-in planning layer: tasks, requirements, blockers, and a kanban board, all stored as graph entities in the same SQLite database. The point is not to replace project management software. It's that a task in atheneum can have edges to the sessions that worked on it, the decisions that shaped it, and the code symbols it touched. Navigation is `atheneum thread` and SQL, not grep on markdown files. A task's full history -- who decided what, which session wrote which file, which agent closed it -- is one graph query away. This is the direction the `session-digest` plan is already pointing toward.

**[agent-envoy](https://crates.io/crates/agent-envoy)** is the HTTP coordination server. It handles agent registration (hierarchical agent IDs, parent-child relationships), structured message passing between agents, session accountability (tool call logging, file write tracking, subagent handoff records), and cross-project symbol search. It's the thing that makes multiple agents aware of each other's existence.

The honest assessment of where this is rough: the polling problem. AI coding assistants speak HTTP request-response. There's no push mechanism. When agent A sends agent B a message, agent B only finds out the next time it explicitly polls. Until MCP adds subscriptions or server-sent events, polling is the only option. The WebSocket endpoint exists but no coding agent uses it natively.

---

## The teaching layer

LLMs are not fine-tuned to use these tools. They don't know what `magellan refs --direction in` does. They don't know the difference between `magellan context impact` and `magellan context affected`. They don't know when to use `mirage cfg` versus `llmgrep search`. Left to their defaults, they fall back to reading files with `cat` and grepping with `rg`, which is exactly the workflow these tools replace.

The fix is [grounded-coding](https://github.com/oldnordic/grounded-coding): a Claude Code plugin that teaches the model to use the stack. It's a set of skills, hooks, and commands that define when and how to invoke each tool. Before touching a symbol, check the graph. Before editing a file, query blast zones. Before claiming something works, run the verification command and read the actual output.

The skill is explicit about the workflow because it has to be. The tools exist; the discipline of using them correctly doesn't come automatically. Without the skill, an agent with access to magellan will still open files directly. With it, the graph tools become the first instinct.

This is not a limitation of the tools. It's a general property of LLM coding assistants: they default to the behavior they were trained on. Changing that default requires explicit instruction. The grounded-coding skill is that instruction layer.

---

## The gap I was trying to close

Tools like LangSmith, LangGraph, Helicone, and Arize Phoenix are serious, well-funded, and do their job well. They are not the target here. The gap this stack tries to close is specific: those tools cost money, require cloud connectivity, and don't connect agent decisions to the code symbols that were affected. For someone building AI coding infrastructure alone, in spare time, with cheap models and no budget, they are out of reach.

The goal is not to be better than LangSmith. It's to offer something that fills a similar role for people who can't pay for it, can't send their code to a cloud service, or need the audit trail to go deeper than LLM call traces. LangGraph handles agent orchestration. What it doesn't do is tell you which code symbols two parallel agents were modifying, whether their blast zones overlapped, or what the call graph looked like at the time of the decision.

That connection -- agent coordination linked to compiler-grade code intelligence -- is what this stack attempts. Not as a replacement for the established tools, but as a free, local-first, GPL-3 complement for the cases those tools don't reach.

The difference matters most in regulated environments. "The LLM suggested it" is not a defensible answer in a financial system audit. A structured audit trail that links agent session to tool call to code symbol to blast zone is. Whether this stack gets there fully is a work in progress. The direction is deliberate.

---

## The numbers as they stand

I'm not going to inflate these. They're real organic download counts, not bots.

| Crate | Downloads | Recent | Version |
|-------|-----------|--------|---------|
| sqlitegraph | 2,988 | 1,775 | 3.3.1 |
| magellan | 1,738 | 1,007 | 4.12.1 |
| splice | 751 | 407 | 2.9.4 |
| llmgrep | 624 | 340 | 3.10.0 |
| mirage-analyzer | 514 | 362 | 1.9.1 |
| geographdb-core | 331 | 331 | 0.5.4 |
| atheneum | 264 | 264 | 0.8.0 |
| agent-envoy | 48 | 48 | 0.3.0 |

20 published crates total, 8,500+ combined downloads. The high recent/total ratio on geographdb-core, atheneum, and agent-envoy means those are new and growing. The higher absolute counts on sqlitegraph and magellan mean they've been accumulating downloads over several months.

magellan is the most mature: 88K LOC, 185 source files, 1,400+ tests, 1,156 commits since December 2025. agent-envoy is the newest: 127 commits, every one of those 48 downloads is recent.

---

## How this was built

Two hours a day. Spare time. Cheap Chinese models and basic-tier frontier models on the occasions I could afford them. No external feedback, no collaborators, no one to tell me whether I was building the right thing.

Before the current generation of AI coding tools got good enough to sustain real development sessions, I used a different approach: write a spec and a plan, then set up cron jobs to trigger automated development runs against that spec overnight. Come back in the morning, review what was generated, fix what was wrong, push forward. It wasn't elegant. It let me ship while sleeping.

That approach produced real data on where autonomous coding workflows fail. The failure modes were consistent:

- **Context loss.** An agent starting a new session knows nothing about what yesterday's run decided. It rediscovers the same facts, re-reads the same files, makes the same mistakes.
- **Scope creep.** Without hard constraints, an agent given a spec for function A will refactor functions B through G "while it's there."
- **Hallucinated success.** An agent with no external verification will report a task complete when it isn't. Confidently. With a summary.
- **File conflicts.** Two tasks touching overlapping code without knowing about each other. The second run silently overwrites the first.
- **Dependency drift.** Long-running sessions accumulate assumed context that's no longer true by the end.

What actually works: small, scoped tasks with explicit success criteria that can be verified by a command whose output you read. One function. One test. One file. Clear pass/fail.

That's shaped the tools. Session continuity (atheneum) exists because context loss was the most common failure. Verification gates (grounded-coding) exist because hallucinated success destroyed more work than any other failure mode. Blast-zone checking (magellan + envoy) exists because file conflicts were invisible. Audit trails exist because "what did the overnight run actually do" was unanswerable without them.

What's interesting now: Claude Code `/loop` and `/goal`, Codex long-horizon planning, and other platforms are converging on the same patterns -- autonomous multi-session workflows, goal-directed agents, structured verification. The problems those features are trying to solve are exactly the failure modes the cron experiments exposed. The difference is that those platforms are building the patterns into the model and product layer. This stack builds them into the infrastructure layer, available independently of which platform you use. Both approaches are necessary. They're not in competition.

The stack was not built during work hours. This matters to say plainly: every commit, every cron run, every morning review happened outside of employment time. I have a day job. None of this is that.

None of this was built without reading other people's work. For the AMD inference side, [hipfire](https://github.com/Kaden-Schutt/hipfire) -- a Rust+HIP inference engine for RDNA GPUs -- was a direct source of insight into how to approach HIP kernel design for this architecture. For the code intelligence side, projects like codeindex and others doing tree-sitter-based indexing showed what the problem space looked like before I started writing my own solution. [Hermes Agent](https://github.com/NousResearch/hermes-agent) (NousResearch) gave insight into how a general-purpose agent handles memory, tool use, and session continuity -- I contributed a small bug fix upstream and learned more from reading the codebase than from the fix itself. Watching how Hermes handles agent evolution and persistent state across interactions influenced several design directions in atheneum and envoy. I'm standing on open-source work. The least I can do is be GPL-3 back.

Building alone without feedback is hard in a specific way. You don't know if something is good until someone else uses it. Download counts are the only signal. The tools have bugs -- they get fixed when found. There are performance areas that need work. The CFG on non-C/Java languages is structural, not precise. Some cross-schema queries don't work yet. These are the honest rough edges of software built by one person in spare time.

That it exists at all, has 8,500+ downloads, and drives my personal coding workflow daily is the part I'm satisfied with.

---

## Where it's going

### Chat navigation and decision capture

The current atheneum graph stores all session data -- tool calls, reasoning logs, decisions -- but querying it requires full scans and per-row JSON extraction. That's the next thing to fix.

The plan adds VIRTUAL generated columns to `graph_entities` that derive `session_id`, `sequence`, and `role` from the existing JSON `data` field. Zero Rust insert-path changes -- the columns are computed on read, not stored. On top of that: a composite index so session queries use the index instead of full scans, and an FTS5 table over chat content so keyword search is sub-millisecond.

Once the schema is right, `atheneum chat` navigates sessions token-budgeted in either direction with role filtering and search. Then a live decision watcher that tails active transcripts and captures structured decisions (from `ExitPlanMode`, `AskUserQuestion`, `TodoWrite` events) as first-class graph nodes in real time.

The goal: answer "what did the agent decide in the last session and why" in under 300 tokens, from graph traversal, without reloading the transcript.

### TUI and web dashboard

The toolset works from the CLI today. The next step is a unified interface: a TUI (ratatui + crossterm) and a companion web dashboard (axum + HTMX) showing everything in one view.

Three panels that matter:
- Live session feed: decisions and tool calls scrolling in real time
- Code intelligence: symbol heat map, most-touched files, blast zone coverage
- Agent coordination: active agents, message queue, concurrent activity on shared files

The web dashboard serves the same data at a shareable URL -- useful for demonstrating the audit trail to a team or showing session history to a stakeholder.

### MCP plugin and swarm coordination

A Claude Code plugin will expose the core tools as MCP endpoints. The surface will be small by design: `magellan_find`, `magellan_refs`, `llmgrep_search`, `mirage_cfg`, `magellan_context`. Six to eight tools covering the most common lookups. The advanced workflows -- token budgets, deep graph traversal, blast-zone analysis -- stay in the skills layer where the full CLI flags are available.

Swarm coordination is the longer-term goal. Before a multi-agent task runs, a coordinator queries blast zones for each planned assignment via `magellan context affected`. The assignments go into a manifest in atheneum. A `PreToolUse` hook checks the envoy swarm registry before any file edit -- if another agent owns that blast zone, the edit is blocked and the coordinator gets a message. After the run, the full DAG (who did what, what they modified, what decisions they made) is queryable from a single `atheneum thread` command.

Two agents editing overlapping code is currently invisible. The goal is to make it a hard gate.

### GNN function ranking

magellan's call graph is a graph. The question I wanted to answer: can a GNN trained on that graph learn which functions are structurally worth improving -- high centrality, high complexity, frequently modified -- and rank them as candidates for automated optimization?

The experiment: extract graph features from magellan (call-graph edges, symbol kinds, CFG complexity, blast-zone size), label 2,474 functions by outcome quality, train a GNN. Result: 0.85 AUC. The model learns to identify promising candidates from graph structure alone.

The intended pipeline connects to [openevolve](https://github.com/openevolve/openevolve), an open-source project (not mine -- credit to its authors) implementing evolutionary program synthesis. openevolve drives the improvement search; the GNN drives the nomination. The full loop:

1. GNN ranks candidate functions from the call graph
2. A subagent creates a local git branch per candidate
3. The GNN's ranked candidate and graph context are pushed to a **local model** (running via ollama or rocmforge) -- no cloud API, no external calls
4. The local model generates the improvement guided by openevolve's evolutionary search
5. Criterion benchmarks run to measure the delta
6. Regression tests and the full test suite validate nothing broke
7. A structured report is written to disk

Step 3 is important: the candidate goes to a local model, not a frontier API. The graph context (call graph, blast zone, CFG complexity) is what makes a smaller local model useful here -- it's not reasoning from scratch, it's operating on structured evidence the GNN already extracted. The size of model needed drops significantly when the nomination and context are precise.

The engineer reviews the report in the morning. Each candidate is isolated on its own branch -- reviewable, discardable, safe to reject. The human stays in the loop at the decision boundary; the autonomous work happens while they're sleeping.

Early experiment. The numbers are real (0.85 AUC on nomination); the full overnight pipeline isn't finished. But the architecture is deliberate: graph evidence for nomination, local inference for privacy and cost, isolated branches for safety, benchmark + test gate for verification, human review for decisions. The same failure modes the cron experiments exposed -- hallucinated success, scope creep, broken tests silently ignored -- are all addressed as hard gates in this loop.

If it works as intended for performance improvements, the same framework extends naturally to security: GNN identifies structurally suspicious functions (high complexity, external inputs, unvalidated paths), the subagent attempts to reproduce a vulnerability, the test suite validates the patch, the report flags it for review. CVE finding and patching in your own codebase, running overnight, with the same branch-isolation and test-gate discipline. Defensive use -- finding your own vulnerabilities before someone else does. That's the direction.

### Storage evolution

sqlitegraph's current HNSW uses standard float32 vectors. The plan is to integrate [TurboVec](https://crates.io/crates/turbovec) -- a Rust crate offering ~8x vector compression via quantization without a training phase -- as the HNSW backing store. At the scale this stack operates (symbol graphs for codebases in the 10K-100K symbol range), that compression changes the index from something that spills to disk to something that fits in L3 cache.

The larger storage work is native-v3: replacing sqlitegraph's B-tree adjacency with CSR (Compressed Sparse Row) layout. B-trees are the wrong data structure for graph adjacency -- they're optimized for range queries, not the "give me all neighbors of node X" pattern that BFS, SCC, and DFS hammer. CSR is cache-optimal for exactly that pattern. Combined with delta-encoded destination IDs (80-90% of edges in a typical codebase are intra-module, meaning small ID deltas) and MVCC on top via an append-only delta log, the hot path becomes SIMD-friendly and fits in cache. The temporal sweep that sqlitegraph 3.3 added maps directly to this -- each checkpoint becomes a CSR snapshot with a delta log on top. One design, two features.

The tiny-pointer compression technique -- recent theoretical work showing that pointer representations in hash structures can be far more compact than 40 years of practice assumed -- has potential to apply to the edge array encoding and possibly to other parts of the stack. This is an experiment, not a shipped feature. The math is interesting enough to pursue.

### Model behavior research

Building a forward-pass tracer into rocmforge was a detour that turned into its own research thread.

The tracer captures every attention edge during inference -- source position, destination position, weight summed across heads and layers -- and emits it as a JSONL stream. I published an [article on Hugging Face](https://huggingface.co/blog/luizspies/technical-xray) covering the initial findings: correct and wrong predictions have measurably different attention graph structures. Wrong predictions show higher attention entropy and more mass on unexpected positions; correct predictions route cleanly to expected context. The model's confidence doesn't distinguish them -- 0.773 for correct, 0.9999 for wrong.

On Qwen2.5-7B with an 80-pair sweep (4,520 trace records), a new pattern emerged: commitment depth varies by task type. Simple pattern-matching tasks (capital cities) commit at L5. Compositional reasoning commits at L13. Visible from the attention graph alone, no probing classifiers needed.

The research question I want to answer next connects directly to the grounded-coding stack: **does providing structured code context (from magellan's symbol graph and call graph) change where a model's attention routes when solving code problems, compared to just throwing file contents at it?**

The hypothesis is that structured graph context changes the problem from retrieval (find the relevant code) to reasoning (understand the relationships), and that this shift might be visible in layer-depth commitment. If it holds, it would give a mechanistic explanation for whether and how the grounded tools change model behavior -- beyond the obvious token-budget argument.

This is future work. rocmforge is alpha. The tracing infrastructure is built; connecting it to real coding sessions with and without the grounded-coding skill is what comes next.

### Local inference

[rocmforge](https://github.com/oldnordic/rocmforge) started from a simple constraint: I have an AMD RX 7900 XT and I wanted to run local inference properly on it. Apple Silicon with unified memory -- the hardware most people use for local LLM work -- is expensive. So I work with what I have.

llama.cpp is excellent and not the target here. Its ROCm/HIP support exists but is built by converting CUDA kernels via `hipify` rather than writing natively for AMD architectures. That's a pragmatic engineering choice for a project that needs to support many backends. The result is that AMD-specific features -- wavefront size, LDS layout, RDNA scheduling characteristics -- get whatever CUDA assumptions get through the conversion layer.

rocmforge is an attempt to write HIP kernels from scratch for AMD hardware: native Q4/Q6 dequantization, KV cache management, attention tracing, measured at 515 tok/s on Qwen2.5-0.5B on the RX 7900 XT. Not competing with llama.cpp. Just trying to understand what my hardware can actually do when you write for it directly, and to learn HIP properly in the process. It's in alpha -- significant work remaining. When it's ready, the last external dependency in the stack closes. Everything runs fully offline: index locally, coordinate locally, run inference locally.

That's the end state: no cloud dependency anywhere in the loop.

---

## Why GPL-3

This wasn't the default choice. I thought about it.

GPL-3 means anyone who ships a product built on this code has to open their modifications. That's the point. It prevents the obvious scenario: a company forks this, closes the modifications, and competes against the open-source version with the benefit of community contributions they don't return.

For individuals and researchers, there's no restriction. Use it, modify it, build on it. It's free.

For regulated industries -- banks, healthcare, government -- the local-first architecture matters more than the license. They cannot send code to a cloud observability service. Their security policies forbid it. This stack runs entirely on-prem, source-visible, auditable down to the SQL schema. If their compliance team asks "what does this tool do to our code", the answer is "read the source, run the queries, check the database file". That's a different posture than "trust our API documentation".

If an organization needs to customize the tools for internal compliance requirements -- specific audit schemas, internal symbol formats, proprietary language support -- they modify the code and use it. If they want modifications maintained upstream, they contribute back. The license enforces that dynamic.

---

## Current state, honest version

**Works and is in daily use:** magellan indexing + watch mode, llmgrep semantic search, mirage CFG analysis (LLVM IR for C/C++, bytecode for Java, AST-based for everything else), splice span-safe edits, sqlitegraph graph traversal + HNSW, atheneum session digest + thread navigation, envoy agent coordination.

**Works, still maturing:** atheneum chat navigation (schema migration in progress), cross-project navigate (symbol search works, edge traversal across schemas has a known bug), envoy cross-project graph walks, CFG precision for AST-backed languages.

**Known rough edges:** bugs exist and get fixed when found. Several performance areas are identified but not yet addressed. The tools are useful now, not finished products.

**Splice specifically needs work.** The core mechanism -- byte-span edits, AST validation, rollback -- is sound. The problem is ergonomics: LLMs skip splice and reach for `write_file` or `multi_edit` instead. Three identified reasons: `splice patch` requires `--db`, `--file`, and `--symbol`, which means three round trips (magellan find, then form the call) before the edit happens. `write_file` is one call. Second: the `complete` subcommand is cursor-based (requires `--line`/`--column`) -- wrong abstraction for an agent that doesn't have a cursor. Third: `--db` is required every time; agents forget or use the wrong path. The improvement plan is written and approved: DB auto-discovery from git root, a `splice edit` text-replace bridge that requires only the text to replace (not the symbol ID), and a `splice suggest` intent-based scaffold generator. Not yet implemented.

**Alpha, significant work remaining:** rocmforge GPU inference, geographdb-core geometric algorithms (the math is sound, the production hardening isn't done).

**Not built yet:** TUI, web dashboard, MCP plugin, swarm coordination, live decision watcher.

The code is at [github.com/oldnordic](https://github.com/oldnordic). Everything is GPL-3. The grounded-coding install script pulls the full stack in one command:

```bash
curl -fsSL https://raw.githubusercontent.com/oldnordic/grounded-coding/master/install.sh | sh
```

If you're building AI coding infrastructure and want something to talk about, I'm reachable on GitHub.
