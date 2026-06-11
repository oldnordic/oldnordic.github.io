---
layout: page
title: Projects
permalink: /projects/
---

Published Rust + Python ecosystem. 14+ crates, 5,500+ combined downloads.

---

## Code Intelligence

**[magellan](https://crates.io/crates/magellan)** — Deterministic code indexing for LLM workflows. Builds a queryable symbol graph from source: call graph, CFG, refs, semantic search (HopGraph via nomic-embed). Used as the grounding layer for agent coding sessions. 1,382+ downloads.

**[llmgrep](https://crates.io/crates/llmgrep)** — Semantic and structural code search over Magellan graphs. Intent-based queries, AST-filtered search, graph traversal.

**[mirage-analyzer](https://crates.io/crates/mirage-analyzer)** — CFG and path analysis. Blast zones, hotspots, dominance frontiers, inter-procedural CFG.

**[splice](https://crates.io/crates/splice)** — Span-safe code editing. Byte-accurate symbol patching and cross-file rename using stable graph IDs.

---

## Agent Infrastructure

**[forge](https://crates.io/crates/forge)** — Rust SDK for AI agent systems. ReAct loop, chat providers (Ollama/OpenAI/Anthropic), tool registry, DAG workflow engine, checkpointing, rollback.

**[envoy](https://crates.io/crates/envoy)** — Agent coordination bus. Session ledger, pub/sub, handoff protocol, multi-agent state.

**[atheneum](https://crates.io/crates/atheneum)** — Knowledge graph for agents. Evidence storage, retrieval, decision logging, wiki/journal ingestion.

---

## Storage and Retrieval

**[sqlitegraph](https://crates.io/crates/sqlitegraph)** — Embedded graph database: HNSW vector search, Cypher-like queries, graph algorithms (BFS, Dijkstra, PageRank, community detection). Python bindings. 2,291+ crates.io downloads, 1,965/month PyPI. BFS 18.3x faster than SQLite adjacency table at 10k nodes.

---

## Local Inference

**[rocmforge](https://crates.io/crates/rocmforge)** — LLM inference engine for AMD GPU (HIP/ROCm). GGUF format, quantized model support. Measured: ~515 tok/s decode on Qwen2.5-0.5B Q4_0, RX 7900 XT. Profiling harnesses included.

---

## Experiments

Language geometry experiments live in [Memoria](https://github.com/oldnordic/Memoria): corpus-native graphs, geometric decoders, curvature analysis. Ongoing, unfinished, results posted as they happen.

---

All download counts from crates.io/PyPI as of June 2026.
