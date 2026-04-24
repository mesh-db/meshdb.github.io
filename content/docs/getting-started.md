---
title: "Getting Started"
date: 2026-04-21
weight: 1
---

MeshDB is a single static binary. Install it, point a Bolt driver at it, and
you have a Cypher-speaking graph database running locally in about a minute.

This page walks through:

1. Installing `meshdb-server`
2. Starting a single-node server
3. Connecting with the official Neo4j Python driver
4. Running your first Cypher query

For cluster mode, authentication, and TLS, see the
[project README](https://github.com/mesh-db/mesh).

## Prerequisites

- **Rust toolchain** (stable) — the easiest install path is
  [`rustup`](https://rustup.rs).
- **`clang` / `libclang-dev`** on Linux. `rust-rocksdb` generates its
  bindings at compile time. On Debian / Ubuntu:

  ```sh
  sudo apt-get install clang libclang-dev
  ```

  On Fedora: `dnf install clang clang-devel`. On macOS the Xcode command
  line tools already ship what's needed.

`protoc` is **not** required — `meshdb-rpc/build.rs` uses a vendored
binary.

## Install

```sh
cargo install meshdb-server
```

The binary lands in `~/.cargo/bin/meshdb-server`. Check the install:

```sh
meshdb-server --version
```

Prefer to build from source? Clone the repo and run:

```sh
git clone https://github.com/mesh-db/mesh.git
cd mesh
cargo build -p meshdb-server --release
```

The binary is then at `./target/release/meshdb-server`.

## Run a single-node server

The smallest useful configuration is a single node with a gRPC listener
and a Bolt listener. You can provide settings via a TOML file, CLI flags,
or environment variables — pick whichever fits your workflow.

### Option A: TOML config

Save as `/tmp/mesh.toml`:

```toml
self_id = 1
listen_address = "127.0.0.1:7001"
data_dir = "/tmp/mesh-data"
bolt_address = "127.0.0.1:7687"
```

Start the server:

```sh
RUST_LOG=info meshdb-server --config /tmp/mesh.toml
```

### Option B: CLI flags

```sh
RUST_LOG=info meshdb-server \
  --self-id 1 \
  --listen-address 127.0.0.1:7001 \
  --bolt-address 127.0.0.1:7687 \
  --data-dir /tmp/mesh-data
```

### Option C: Environment variables

Every flag has a matching `MESHDB_*` env var:

```sh
MESHDB_SELF_ID=1 \
MESHDB_LISTEN_ADDRESS=127.0.0.1:7001 \
MESHDB_BOLT_ADDRESS=127.0.0.1:7687 \
MESHDB_DATA_DIR=/tmp/mesh-data \
RUST_LOG=info meshdb-server
```

When `--config` and flags are both present, the TOML file is loaded first
and any set flags override the matching fields. Structured settings
(`peers`, `bolt_auth`, `bolt_tls`, `grpc_tls`) stay TOML-only.
Run `meshdb-server --help` for the full flag list.

Whichever option you pick, you should see:

```
INFO meshdb_server: starting meshdb-server self_id=1 listen_address=127.0.0.1:7001 data_dir=/tmp/mesh-data peers=0
INFO meshdb_server: meshdb-server listening addr=127.0.0.1:7001
INFO meshdb_server: meshdb-server bolt listening addr=127.0.0.1:7687
```

Leave it running. `Ctrl-C` to stop. Delete `/tmp/mesh-data` between
runs if you want a clean slate.

`7001` is the gRPC port (cluster-internal, client reads and writes via
the gRPC surface when you skip Bolt). `7687` is the standard Neo4j Bolt
port — every driver defaults to it.

## Connect from a Bolt client

Any Neo4j-compatible driver works. The official Python driver is the
quickest way to verify the server:

```sh
python -m pip install --user neo4j
```

Save as `/tmp/mesh_demo.py`:

```python
from neo4j import GraphDatabase

# In the default config auth is accepted-but-ignored — any credentials work.
driver = GraphDatabase.driver("bolt://127.0.0.1:7687", auth=("any", "any"))

with driver.session() as s:
    # Auto-commit with parameters.
    s.run("CREATE (n:Person {name: $name, age: $age})", name="Ada", age=37)
    s.run("CREATE (n:Person {name: $name, age: $age})", name="Grace", age=85)

    # Parameterized read.
    result = s.run(
        "MATCH (n:Person) WHERE n.age > $min RETURN n.name AS name, n.age AS age ORDER BY age",
        min=40,
    )
    for record in result:
        print(record["name"], record["age"])

    # Explicit transaction — both creates land atomically on commit.
    with s.begin_transaction() as tx:
        tx.run("CREATE (n:Project {title: $title})", title="mesh")
        tx.run("CREATE (n:Project {title: $title})", title="bolt-listener")

    # UNWIND with a list parameter — canonical batch idiom.
    s.run("UNWIND $items AS x CREATE (:Tag {name: x})",
          items=["rust", "graph", "cypher"])

    print("--- all tags ---")
    for r in s.run("MATCH (n:Tag) RETURN n.name AS name ORDER BY name"):
        print(r["name"])

driver.close()
```

Run it:

```sh
python /tmp/mesh_demo.py
```

Expected output:

```
Grace 85
--- all tags ---
cypher
graph
rust
```

That single script exercises the Bolt handshake, PackStream encoding,
parameter binding, pattern-property parameters, explicit transactions,
and `UNWIND $list` — the full driver-facing surface.

## Your first graph in Cypher

Inside any session you can write regular openCypher. A minimal social
graph and a reverse-lookup query:

```cypher
CREATE (a:Person {name: "Ada"})-[:KNOWS]->(b:Person {name: "Grace"})
RETURN a, b;
```

```cypher
MATCH (p:Person)-[:KNOWS]->(friend)
RETURN p.name AS person, collect(friend.name) AS friends
ORDER BY person;
```

MeshDB supports a broad openCypher surface: `MATCH` / `OPTIONAL MATCH`,
`WHERE`, `RETURN` (with `DISTINCT`), `WITH`, `ORDER BY` / `LIMIT` / `SKIP`,
`UNION`, `CREATE`, `MERGE` (with `ON CREATE SET` / `ON MATCH SET`),
`SET`, `REMOVE`, `DELETE` / `DETACH DELETE`, variable-length paths,
Neo4j 5 quantifier shorthand (`->+`, `->*`, `->{n,m}`), `shortestPath`,
list comprehensions, pattern comprehensions, `CASE`, `EXISTS { ... }`,
`COUNT { ... }`, `COLLECT { ... }`, `CALL { ... }` subqueries,
`UNWIND`, `FOREACH`, `LOAD CSV` — plus the full scalar-function and
aggregate surface you'd expect from Neo4j.

**Schema.** `CREATE INDEX` / `DROP INDEX` / `SHOW INDEXES` for node
(`FOR (n:Label) ON (n.a, n.b)`) and relationship
(`FOR ()-[r:TYPE]-() ON (r.p)`) scopes, including composite keys that
the planner rewrites pattern-property equalities and `WHERE` conjuncts
against. `CREATE POINT INDEX` backs `point.withinbbox` and
`point.distance` queries on `Property::Point` columns with a Z-order
(Morton) quantizer — Cartesian and WGS-84 (geographic) coordinates
both index. `CREATE CONSTRAINT` / `DROP CONSTRAINT` /
`SHOW CONSTRAINTS` for `UNIQUE`, `NOT NULL`, `IS :: <TYPE>`, and
composite `IS NODE KEY`, on node or relationship scope. DDL replicates
across Raft and routing clusters.

**Transactions.** Bolt `BEGIN` / `COMMIT` / `ROLLBACK` are fully wired
— multi-statement transactions accumulate their writes and commit
atomically through the same single-node / Raft / routing-2PC
machinery as auto-commit RUNs, with read-your-writes overlay between
RUNs in the same transaction. For large bulk writes that shouldn't
land in one transaction,
`CALL { ... } IN TRANSACTIONS [OF n ROWS] [ON ERROR { FAIL | CONTINUE
| BREAK }]` splits a stream across independently-committed batches.

**APOC.** MeshDB ships an APOC-compatible surface in the
`meshdb-apoc` crate, enabled by default. Included today:
`apoc.coll.*`, `apoc.text.*`, `apoc.map.*`, `apoc.util.*` (hashes),
`apoc.convert.*` (JSON), `apoc.date.*`, `apoc.number.*`,
`apoc.create.*` (both scalars and write procedures),
`apoc.refactor.setType`, `apoc.meta.*`, `apoc.agg.*` aggregates,
and `apoc.periodic.iterate(iterateQuery, actionQuery, config)` as a
planner-level rewrite over the batched-commit dispatcher. Embed
callers who want a slimmer binary can opt out per-namespace with
`--no-default-features`.

For the complete feature inventory and known limitations, see
[What works](https://github.com/mesh-db/mesh#what-works) in the
project README.

## Next steps

- **Lock down the Bolt listener.** The default accepts any credentials —
  fine for local dev, not for anything shared. Add a
  [`[[bolt_auth.users]]`](https://github.com/mesh-db/mesh#bolt-authentication)
  block (plain text or bcrypt) to require authentication.
- **Add TLS.** Both the Bolt listener
  ([`[bolt_tls]`](https://github.com/mesh-db/mesh#bolt-tls)) and the
  cluster-internal gRPC surface
  ([`[grpc_tls]`](https://github.com/mesh-db/mesh#grpc-tls)) can
  terminate TLS directly.
- **Go multi-peer.** Set `mode = "raft"` for a fully-replicated cluster,
  or `mode = "routing"` for hash-partitioned sharding with 2PC. See
  [Cluster mode](https://github.com/mesh-db/mesh#cluster-mode-multi-peer)
  for worked three-peer configs.
- **Browse the source.** MeshDB is MIT-licensed on
  [GitHub](https://github.com/mesh-db/mesh). Issues and discussions
  are welcome.
