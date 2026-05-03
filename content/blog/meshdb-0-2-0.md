---
title: "MeshDB 0.2.0"
date: 2026-04-26
draft: false
---

MeshDB 0.2.0 is out. The headline is operational hardening — the
knobs and surfaces you need to actually run a cluster: graceful
drain, k8s probes, per-query budgets, plan cache, consistent backup
with offline verification, TLS cert hot-reload, mTLS, OpenTelemetry,
an audit log, and a full multi-raft mode. There are no breaking
changes to the on-disk format, network protocols, or config shape;
upgrade is a process restart on the same data directory.

Install the latest release with:

```sh
cargo install meshdb-server
```

The reference container image is `darkspar/meshdb-server:0.2.0` on
Docker Hub. The new
[`deploy/`](https://github.com/mesh-db/meshdb/tree/main/deploy) tree
in the repo ships production-shaped Kubernetes manifests, a
Prometheus alert bundle, an importable Grafana dashboard, and a
day-2 [`RUNBOOK.md`](https://github.com/mesh-db/meshdb/blob/main/deploy/RUNBOOK.md).

## Multi-Raft mode

`mode = "multi-raft"` is the third cluster shape: sharding *with*
per-partition replication. Each partition has its own openraft group
replicated across `replication_factor` peers; DDL and cluster
membership ride a separate metadata Raft group spanning every peer.
Cross-partition writes use a Spanner-style 2PC where both PREPARE
and COMMIT are proposed through the partition Raft, so staged state
is replicated by the time PREPARE-ACK returns and there's no
in-doubt window dependent on a participant log fsync.

The mechanics that make it useful in practice:

- **Server-side proxying.** Single-partition writes arriving on a
  non-leader peer are proxied to the partition leader via internal
  `MeshWrite::ForwardWrite`; DDL through `MeshWrite::ForwardDdl`.
  Bolt clients see one consistent endpoint for the lifetime of a
  session, no client-visible redirects.
- **DDL barrier.** Every peer tracks the highest meta-Raft index
  it's seen committed; partition writes await the local meta replica
  to catch up before applying, so a `CREATE INDEX` issued through
  any peer is guaranteed visible by the time a follow-up write
  lands.
- **Per-partition snapshots.** Each partition's applier packs only
  its own nodes and edges (~1/N of cluster data), so a new replica
  catching up via `InstallSnapshot` doesn't download the full graph.
- **Dynamic rebalancing.** `add_partition_replica` /
  `remove_partition_replica` wrap openraft's per-partition
  `change_membership`; `instantiate_partition_group` spins up a new
  partition Raft + RocksDB dir + applier on a peer that didn't
  bootstrap with that partition, no restart required. The cluster's
  persisted view of placement (`PartitionReplicaMap`) updates
  atomically via `ClusterCommand::SetPartitionReplicas`.
- **Linearizable reads.** Scatter-gather reads in multi-raft and
  routing modes serialize through the partition leader for
  linearizability; learner / read-replica APIs let operators add
  non-voting replicas for read fan-out.
- **Periodic 2PC recovery.** A 60s loop re-resolves any in-doubt
  PREPAREs left by a coordinator that crashed mid-flight while the
  cluster stayed up.

## Operational hardening

The full new surface, all configurable through TOML:

- **Graceful drain on SIGTERM / SIGINT.** Every partition this peer
  leads steps down before the listener exits. Bounded by
  `shutdown_drain_timeout_seconds` (default 30). Pairs with k8s
  `terminationGracePeriodSeconds` for zero-drop rolling restarts.
- **K8s liveness / readiness probes.** `/livez` and `/readyz` on
  the metrics endpoint. The shutdown path flips readiness *before*
  draining, so the kubelet pulls a peer out of rotation before it
  stops accepting traffic.
- **Per-query budgets.** `query_timeout_seconds` (deadline on every
  Cypher RUN), `query_max_rows` (cap on result-set size,
  `ResourceExhausted` instead of OOM), `max_concurrent_queries`
  (per-peer concurrency cap).
- **Cypher plan cache.** `plan_cache_size = N` skips parse + plan
  on repeated parametrised queries. Self-correcting on every peer
  when DDL changes the schema — entries are keyed on a fingerprint
  hashed from the local index registry.
- **Cluster-wide consistent backup.** `MeshService::take_cluster_backup`
  snapshots every Raft group across every peer in parallel; the
  manifest serializes to JSON. The new
  `meshdb-server validate-backup --manifest=path --data-dir=path --peer-id=N`
  CLI subcommand confirms a restored data dir matches the manifest
  before you bring the cluster back up.
- **TLS cert hot-reload.** Set `[bolt_tls] reload_interval_seconds`
  and / or `[grpc_tls] reload_interval_seconds` and the listener
  swaps in rotated certs without restarting. Pairs with
  cert-manager / ACME pipelines.
- **gRPC mTLS.** `[grpc_tls] client_ca_path` requires every inbound
  TLS handshake to present a client cert chained to the bundle.
  Outbound peer endpoints automatically present the configured
  identity.
- **Cluster auth.** `[cluster_auth] token` adds a shared-secret
  bearer token on every inter-peer and admin RPC, validated in
  constant time.
- **OpenTelemetry / OTLP.** `[tracing] otlp_endpoint = "..."`
  exports every `tracing::span` (gRPC handlers, Raft applies, the
  Cypher executor) through the OpenTelemetry SDK's batch span
  processor. `service_name` and `sample_rate` are operator-tunable.
- **Audit log.** `audit_log_path` writes one fsync'd JSONL record
  per admin operation (drain, backup, …) with timestamp + initiator
  + structured args + success/error.
- **RocksDB tuning.** A new `[storage]` section exposes
  `max_open_files`, `write_buffer_size_bytes`,
  `max_write_buffer_number`, `bloom_filter_bits_per_key`,
  `keep_log_file_num`. Defaults match the historical hand-applied
  values, so existing deployments see no behavior change unless
  they opt in.

## Cypher: `allShortestPaths` locked in

`allShortestPaths(...)` is now a supported planner + executor
surface. The shared layered-BFS operator builds a parent DAG that
records every `(parent, edge)` pair at each shortest-path level,
and the reconstruction walk enumerates every minimum-length path
(vs. `shortestPath(...)`'s single-path early exit). Cycles, parallel
edges, undirected patterns, reverse direction, edge-type unions,
and self-loops are all covered by integration tests. The capability
was present in code but documented as deferred; this release sheds
the deferral and locks in the behavior with paranoia tests.

## Production deployment artifacts

The new [`deploy/`](https://github.com/mesh-db/meshdb/tree/main/deploy)
tree:

- `k8s/` — ConfigMap with operational knobs set, headless + Bolt
  ClusterIP services, a 3-pod StatefulSet with anti-affinity,
  OrderedReady, and livez/readyz probes, plus a PodDisruptionBudget
  (max 1 unavailable).
- `grafana/meshdb-cluster.json` — importable dashboard: query rate,
  p99 latency, leader skew, apply lag, in-doubt PREPAREs, DDL gate,
  forward writes.
- `prometheus/alerts.yaml` — peer-down, meta-without-leader, apply
  lag, in-doubt PREPAREs, DDL gate timeout, sustained p99 > 1s.
- `RUNBOOK.md` — day-2 ops: rolling upgrades, drain, backup/restore,
  TLS rotation, adding/removing peers, common pitfalls.

## Upgrading from 0.1.0

No action required beyond a binary restart. No data migration, no
config changes, no protocol changes. The `[storage]`,
`[cluster_auth]`, `[tracing]`, `audit_log_path`,
`shutdown_drain_timeout_seconds`, `query_timeout_seconds`,
`query_max_rows`, `max_concurrent_queries`, and `plan_cache_size`
fields are all additive — leave them unset and the server behaves
exactly as it did in 0.1.0.

## What's next

- Auto leader balancer (blocked on openraft 0.9 exposing
  `transfer_leader` / TimeoutNow).
- Audit-log coverage of the cluster-level paths
  (`add_partition_replica`, `drain_peer`, learner ops) — the
  service-level paths land first; the cluster-level paths are a
  mechanical follow-up.
- Pull-based result streaming through Bolt PULL — `query_max_rows`
  is the bound today; the executor still materializes rows in
  `Vec<Row>` before the response goes out.
- A CI perf-regression gate for the criterion benches that shipped
  in 0.1.0.
- `meshdb-server backup` and `meshdb-server cluster add-peer` CLI
  subcommands so backup-trigger and peer-add aren't gRPC-only.
