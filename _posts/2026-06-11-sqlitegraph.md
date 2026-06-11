---
layout: post
title: "SQLiteGraph: Why I Built One Embedded Library Instead of Four Services"
date: 2026-06-11
categories: engineering
---

I needed a graph database for a code intelligence project. The typical recommendation would have been Neo4j, or at minimum a graph library backed by something like PostgreSQL. But I was building a CLI tool that runs as a single process on a developer's laptop. Pulling in a graph database server, a vector search service, a KV cache, and a pub/sub bus — plus connection management and process lifecycle for each — is the wrong architecture for that constraint.

So I built [sqlitegraph](https://github.com/oldnordic/sqlitegraph): a single Rust crate that embeds all four of those things, backed by SQLite, with no server required.

This post covers what it does, how it works, the bugs I hit, and where the tradeoffs are. It's not a comparison to production graph databases — that's not what it's for.

---

## The problem it solves

The tools I'm building ([magellan](https://crates.io/crates/magellan), [llmgrep](https://crates.io/crates/llmgrep), [splice](https://crates.io/crates/splice), [mirage-analyzer](https://crates.io/crates/mirage-analyzer)) all need the same set of primitives:

- **Graph storage and traversal** — symbol relationships, call graphs, import chains
- **Vector search** — embedding-based semantic lookup (HNSW)
- **Key-value store** — index metadata, cached results, cross-process state
- **Pub/sub** — live change notification during file watching

With separate services that's SQLite + Qdrant (or Pinecone) + Redis + some message bus. Each of those is a process to manage, a port to configure, a connection pool to tune, a service to restart when it crashes. And cross-domain queries — "find nodes whose vector embedding is similar to X and that have outgoing edges to Y" — become application-level joins over separate API calls.

With sqlitegraph it's one `.db` file and one dependency.

---

## What it does

### Graph layer

Nodes and edges with typed kinds. Nodes carry a `kind` (string), `name`, and arbitrary JSON data. Edges carry a `kind` and an optional float weight.

```rust
use sqlitegraph::backend::{GraphBackend, NodeSpec, EdgeSpec};
use sqlitegraph::backend::sqlite::SqliteGraphBackend;

let backend = SqliteGraphBackend::open("graph.db")?;

let alice = backend.insert_node(NodeSpec {
    kind: "User".to_string(),
    name: "Alice".to_string(),
    file_path: None,
    data: serde_json::json!({"age": 30}),
})?;

let bob = backend.insert_node(NodeSpec {
    kind: "User".to_string(),
    name: "Bob".to_string(),
    file_path: None,
    data: serde_json::json!({"age": 31}),
})?;

backend.insert_edge(EdgeSpec {
    source_id: alice,
    target_id: bob,
    edge_type: "KNOWS".to_string(),
    weight: None,
    data: serde_json::json!({}),
})?;
```

### Graph algorithms

35+ algorithms implemented ([full list in the README](https://github.com/oldnordic/sqlitegraph#graph-algorithms)): BFS, DFS, k-hop, shortest path, PageRank, betweenness centrality, SCC (Tarjan), topological sort, Louvain community detection, label propagation, cycle search, dominator trees, control dependence, taint analysis, subgraph isomorphism, critical path, and more.

```rust
let neighbors = backend.bfs(alice, 2, sqlitegraph::Direction::Outgoing)?;
let components = backend.strongly_connected_components()?;
let communities = backend.louvain(100)?;
```

### Cypher-inspired query language

3.0 added a query engine. Not full Cypher — it covers the subset I needed.

```
MATCH (a:User)-[:KNOWS]->(b:User) RETURN a.name, b.name
MATCH (n:Function) WHERE n.complexity > 20 RETURN n.name
MATCH (a)-[:CALLS*1..3]->(b) WHERE a.name = "main" RETURN b.name
CALL db.index.vector.queryNodes('embeddings', 5, [1.0, 0.8, 0.1])
```

Supports: node/edge patterns, label filtering, property filtering with `AND`/`OR` and parentheses, variable-depth hops, `LIMIT`, `RETURN`, `CREATE`, `SET`, `DELETE`, and the `CALL` form for k-nearest-neighbor vector queries.

From Python:

```python
from sqlitegraph import Graph

g = Graph.open("graph.db")
result = g.query("MATCH (a:User)-[:KNOWS]->(b:User) RETURN a.name, b.name")
```

From CLI:

```bash
sqlitegraph --db graph.db query "MATCH (n:Function) WHERE n.name =~ 'parse.*' RETURN n.name"
sqlitegraph --db graph.db algo scc
sqlitegraph --db graph.db algo louvain --max-iterations 100
```

### HNSW vector search

Hierarchical Navigable Small World graphs for approximate nearest-neighbor search. Multiple named indexes per database, persistent across sessions, three distance metrics (cosine, dot product, Euclidean).

```python
g = Graph.open("graph.db")

idx = g.create_hnsw_index("code-embeddings", dimension=768, metric="cosine")
idx.insert_vector([...], {"symbol": "parse_token", "file": "lexer.rs"})
idx.insert_vector([...], {"symbol": "lex_identifier", "file": "lexer.rs"})

results = idx.search(query_vector, k=5)
```

The SIMD distance kernels auto-detect the highest available instruction set at runtime (AVX-512F → AVX2 → scalar). Measured on Ryzen 7800X3D:
- Cosine similarity at 1536 dims: AVX-512 is ~30x scalar, ~3x AVX2
- Euclidean distance: AVX-512 is ~2–3x AVX2

### Two backends

**SQLite backend** (default): standard `.db` file. Stable, inspectable with any SQLite tool, scales well at larger graph sizes. The mature B-tree indexing and mmap give it good behavior at 50K+ nodes.

**Native V3 backend**: custom B+Tree page store, `.graph` file. Adds LRU cache, parallel BFS, and pub/sub. Faster than SQLite for warm-cache point lookups (~27x at single-record reads). Slower than SQLite at large-scale traversals. Still included in the crate, but for the codebase sizes in my stack SQLite covers the workload so I default to it.

From the 2026-06-07 benchmarks on Ryzen 7800X3D:

| Benchmark | SQLite | Native V3 |
|-----------|--------|-----------|
| BFS 1K nodes / 5K edges | 2.37 ms | 3.32 ms |
| BFS 10K nodes / 50K edges | 26.5 ms | 56.2 ms |
| Point lookup (warm cache) | 3965 ns | 146 ns |

SQLite wins on traversal workloads. V3 wins on warm-cache point lookups. For code intelligence, most of my workloads are traversal-heavy, so SQLite is the default.

---

## The bugs that had to be fixed

The changelog tells the real story of what broke during development.

**HNSW topology not persisted (v3.0.x series)**

The initial persistent HNSW implementation stored vectors in SQLite but didn't save the graph topology (neighbor lists, entry points, layer assignments). Each session had to rebuild the entire HNSW graph from scratch — O(N log N) for every open. At 126K vectors, this was unusable.

Fixed in 3.1.0: `persist_topology()` and `restore_topology()` write/read the layer structure to SQLite. Reopen time dropped from O(N log N) rebuild to O(E) restore.

**Bulk persist was O(N²) (v3.2.4)**

After fixing persistence, `persist_topology()` was still deleting every `hnsw_layers` record and re-inserting all of them on every single vector insert. For the rocmforge codebase (~126K layer records), this produced ~2.5 vectors/second. Expected throughput is ~30 vectors/second.

Fixed by adding a `dirty_nodes: HashSet<u64>` to `HnswLayer`. Only modified nodes get written. The hotpath is now O(modified nodes), not O(all nodes).

**Sequential ID assumption in multi-index mode (v3.1.5)**

The original `search()` loaded vectors as a `Vec<Vec<f32>>` indexed by `vector_id - 1`. This assumes IDs start at 1 and are sequential — which breaks as soon as you have a second HNSW index in the same database (IDs start where the previous index left off) or after any delete+recreate cycle.

Fixed by switching to `HashMap<u64, Vec<f32>>` keyed by node ID, and building a per-level map during search so layers with independent ID spaces don't interfere.

**Bulk insert doing O(N) fsyncs (v3.2.1)**

`batch_insert_vectors()` was calling `store_vector()` in a loop. Each `store_vector()` was an individual autocommit INSERT — an O(N) fsync sequence. Wrapping the loop in a single `BEGIN IMMEDIATE` / `COMMIT` transaction made it O(1) fsyncs.

**B+Tree split at 100K+ nodes (v2.1.0)**

`MIN_KEYS` was set to 126 instead of 125. The off-by-one in the split index calculation meant B+Tree splits were invalid, causing panics when the node count crossed certain thresholds. Fixed and verified with a stress test at 100K+ inserts.

---

## How it's used

The primary consumer is the code intelligence toolchain. Magellan uses sqlitegraph as its storage backend — every parsed symbol, call graph edge, and semantic embedding lives in a sqlitegraph database. llmgrep queries that same database for semantic search. mirage-analyzer traverses it for CFG and dominance analysis.

The HNSW index backs magellan's HopGraph — embedding-based semantic symbol search. You give it a natural language query, it returns symbols by vector similarity, then follows graph edges to retrieve callers and callees.

At the current scale (all 9 code intelligence projects indexed), the database holds roughly 210K+ symbols and 37K+ embedding vectors. The SQLite backend handles this without issue.

---

## What it's not

Not a replacement for Neo4j, ArangoDB, or Qdrant at production scale. Those systems are distributed, have cluster replication, operational tooling, and years of optimization work behind them. SQLiteGraph is embedded — it runs in-process, in a single file, with no server. It's the right tool for CLI tools, local agents, and developer tooling. It's not the right tool for a multi-tenant SaaS database.

---

## How to reproduce

```bash
git clone https://github.com/oldnordic/sqlitegraph
cd sqlitegraph

# Run the benchmark suite
./scripts/run-curated-benchmarks.sh

# Python demo (requires uv)
pip install sqlitegraph
python -c "
from sqlitegraph import Graph
g = Graph.open_in_memory()
a = g.add_node(kind='User', name='Alice', data={'age': 30})
b = g.add_node(kind='User', name='Bob', data={'age': 31})
g.add_edge(a, b, 'KNOWS')
print(g.query(\"MATCH (a:User)-[:KNOWS]->(b:User) RETURN a.name, b.name\"))
"

# CLI demo
cargo install sqlitegraph-cli
sqlitegraph --db /tmp/demo.db --write insert --kind User --name Alice --data '{"age":30}'
sqlitegraph --db /tmp/demo.db --write insert --kind User --name Bob --data '{"age":31}'
sqlitegraph --db /tmp/demo.db --write query 'CREATE (1)-[:KNOWS]->(2)'
sqlitegraph --db /tmp/demo.db query 'MATCH (a:User)-[:KNOWS]->(b:User) RETURN a.name, b.name'
```

Hardware used for benchmarks: AMD Ryzen 7 7800X3D, 64 GB RAM, tmpfs. Rust 1.95.0.

Downloads: 2,291+ on crates.io, 1,965/month on PyPI as of June 2026.
