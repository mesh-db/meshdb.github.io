---
title: "MeshDB 0.1.0"
date: 2026-04-24
draft: false
---

MeshDB 0.1.0 is out. This is the first stable minor cut from the
alpha line, and it comes with no breaking changes to the on-disk
format, network protocols, or config shape from `0.1.0-alpha.9` —
upgrade is a process restart on the same data directory.

Install the latest release with:

```sh
cargo install meshdb-server
```

## Highlights

**Distributed 2PC is now test-verified**, not just documented. Four
new failure-injection integration tests cover coordinator crash
pre-decision (restart drives `ABORT`), coordinator crash mid-COMMIT
fanout (restart resumes `COMMIT` idempotently), participant crash
after `PREPARE`-ACK (cross-peer `ResolveTransaction` recovery), and
PREPARE-rejection synchronous rollback. Deterministic injection
rides on a cfg-gated `FaultPoints` struct — zero cost in release
builds.

**`apoc.periodic.iterate` now honors `parallel: true` /
`concurrency: N`.** Batches dispatch through a
`tokio::sync::Semaphore`-bounded worker pool with
`Arc<Mutex<...>>`-guarded stats accumulators. Throughput wins
materialize fully in single-node mode; Raft and routing-2PC modes
still serialize commits internally, so the gain there is limited to
hiding per-batch planning and read latency.

**Criterion benches for storage and query hot paths** shipped under
`crates/meshdb-storage/benches/` and `crates/meshdb-rpc/benches/`.
Eleven benches cover point lookup, single and batched writes,
adjacency scans, label and indexed seeks, and end-to-end Cypher
(`CREATE`, `MATCH`, one-hop, `count`). Local-run only — no CI
regression gating yet.

**Public error enums are `#[non_exhaustive]`.** Future variants on
`meshdb-core::Error`, `meshdb-storage::Error`, `meshdb-cypher::Error`,
`meshdb-executor::Error`, `meshdb-cluster::Error`,
`meshdb-bolt::BoltError`, `meshdb-rpc::ConvertError`, and
`meshdb-apoc::ApocError` can be added without a breaking-change bump.

## Stability commitments for 0.x

- **Bolt wire protocol (v4.4–5.4):** stable within a minor; breaking
  additions cut a new minor.
- **gRPC internal protocol (`proto/mesh.proto`):** intra-cluster
  only; treated as stable within a minor but no external guarantee.
- **On-disk data layout (RocksDB column families):** breaking
  changes will call out a migration path in release notes. No
  breaks in `0.1.x`.
- **TOML config schema:** additive within a minor; fields may be
  added, none removed. Unknown fields continue to error via serde's
  `deny_unknown_fields`.
- **Public Rust API:** error enums may grow variants at any point
  (hence the `#[non_exhaustive]` annotation); other public types may
  break at minor bumps (`0.2`, `0.3`, etc.) until 1.0 crystallizes
  the surface. Consumers embedding MeshDB as a library should pin to
  exact minor versions.

## Known limitations deferred past 0.1.0

- `allShortestPaths` parses but the planner rejects it. Single
  `shortestPath(...)` works.
- LDG streaming partitioner is not implemented — only the FNV-1a
  hash partitioner. Adequate for uniform workloads; skew-sensitive
  workloads will want the streaming partitioner.
- Vectorized / columnar execution is out of scope. The current
  Volcano/iterator model is well-suited to OLTP; analytical
  workloads at 100M+ rows will eventually want a vectorized runtime.
- `apoc.periodic.iterate` parallel gains in Raft / routing-2PC modes
  are limited by internal serialization of the commit path.
  Single-node mode gets the full throughput benefit.
- Criterion benches run locally only. A CI perf-regression gate
  (baseline + variance budget) is a follow-up.

## Upgrading from `0.1.0-alpha.9`

No action required beyond a binary restart. No data migration, no
config changes, no protocol changes. Alpha releases have been wire-
and storage-compatible with 0.1.0 throughout.
