# Cache Sync Agent: Cross-Cache-Server Data Replication

## Summary

The cache-server is today a singleton in a kcp installation, making it a single point of
failure. This proposal removes that constraint by introducing the **cache-syncer**,
a per-cache-server component that replicates shard data across all cache-server instances.
With it, kcp can run multiple cache-servers for "higher" availability: the loss of any one
instance does not interrupt shards connected to the others, and any instance can serve global
list/watch requests independently. Capacity scaling — partitioning data so each
instance stores only a subset of the total — is out of scope and requires a follow-up.

## Motivation

Each shard is configured with a kubeconfig pointing to exactly one cache-server (`--cache-kubeconfig`),
and all shards are expected to share that same instance. This makes the cache-server a **single point of
failure** — if it becomes unavailable, every shard in the installation loses its ability to
replicate data, and any global consumer (e.g. a controller performing a cross-shard list/watch)
loses visibility into the global kcp state. The only redundancy available today is what the
cache-server's etcd provides, which does not help if the cache-server process itself is
down or unreachable.

This proposal enables kcp to run **more than one cache-server instance**, so that the cache tier
gains the availability properties expected of a production-grade control plane component. Each
cache-server operates on its own backing store. Shards continue to push data to exactly one
cache-server, but a new component — the **cache-syncer** — replicates that data across all peer
instances. Any cache-server in the mesh can then serve global list/watch requests, and the loss
of a single instance no longer interrupts service for the rest of the installation.

**Capacity** is explicitly out of scope for this proposal. Running multiple cache-servers with
full data replication does not reduce the data stored per instance; sharding the cache tier so
that each instance only stores a subset of the total data is a separate problem that requires a
follow-up design.

### Goals

1. The cache tier must tolerate the loss of a single cache-server instance without interrupting
   shards connected to other instances.
2. Each cache-server must eventually hold a complete copy of all shard data from every peer, so
   any instance can serve global list/watch requests independently.
3. Peer discovery must be fully automated via existing Shard and new Cache objects — no manual peer
   list maintenance after initial bootstrap.
4. Data replication must be event-driven; the steady-state cost must not grow with shard count
   beyond the known informer-count bound.
5. No changes to the existing shard-to-cache-server push path in kcp shards.
6. Shard data must be written to peer cache-servers using the same etcd key structure
   (`<StoragePrefix>/<Group>/<Resource>/.../<Shard>/<Cluster>/...`) as if the shard had pushed
   it directly.

### Non-Goals

1. **Capacity scaling.** Replication produces a full copy of all data on every instance; reducing
   per-instance storage by partitioning data across cache-servers is a follow-up.
2. Strong consistency. Eventual consistency is the correctness model; there is no hard bound on
   propagation delay.
3. Shard decommissioning or garbage-collection of stale shard data from cache-servers. This is
   deferred: the kcp-side workflow for shard removal is not yet ready.
4. Modification of the in-shard replication reconcilers (`pkg/reconciler/cache/replication`).
5. Federation of kcp root clusters or cross-installation replication.

## Proposal

### Topology

One cache-syncer is deployed per cache-server. Each syncer is responsible for replicating
data from its **source** cache-server into all other known cache-servers (**peers**).

A cache-server holds the **authoritative** copy of data for a shard if that shard's Shard object
carries `kcp.io/cache` equal to that cache-server's name. The sync topology is star-shaped:
shards connected to cache-server A are synced into B, C, D by A's syncer; shards connected to
B are synced into A, C, D by B's syncer; and so on. Write conflicts between syncers cannot
occur because each shard is authoritative on exactly one cache-server.

### Configuration

The cache-syncer accepts the following flags:

| Flag | Description |
|---|---|
| `--peer-ca-file` | CA certificate used to verify peer cache-server TLS. |
| `--peer-cert-file` | Client certificate used to authenticate to peers. |
| `--peer-key-file` | Key for the client certificate. |
| `--initial-peer-urls` | Comma-separated peer cache-server URLs for bootstrapping the peer mesh before peer Cache objects are discovered via shard data. Optional; not needed in a single-cache-server installation. |

### Identity

On startup, the syncer reads the Cache object from the source's `system:cache:server` shard
and `system:cache` cluster. That location holds exactly one object — the cache-server's own — so
no annotation is needed to disambiguate it. The object's name becomes the syncer's own
cache-server name, used throughout to filter authoritative shards.

### Peer Discovery

Peer connectivity is bootstrapped and maintained as follows.

**Bootstrap (one-time):** For each URL in `--initial-peer-urls`, the syncer performs a
cross-shard LIST (`*` shard, `system:shard` cluster) of Cache objects on that peer. This seeds
the initial peer list. Because Cache objects from other cache-servers propagate through shard data
(see Propagation below), a single initial peer URL is sufficient to discover the full existing
mesh transitively.

**Ongoing discovery:** The syncer maintains a cross-shard watch of Cache objects on its
source. When a new Cache object appears, the syncer reads `spec.baseURL`, constructs a client
using the `--peer-*` credentials, and begins syncing to that peer.

**Propagation:** Each connected shard pulls the cache-server's Cache object from
`system:cache:server/system:cache` into its `system:shard` cluster during bootstrap (existing
behaviour in `pkg/reconciler/core/cache`). Because shard data is synced across all peers by each
respective syncer, Cache objects propagate through the mesh without a dedicated reconciler —
they travel as ordinary shard data.

**Example — cache-c joins an existing two-server mesh (cache-a, cache-b):**

1. cache-c starts with `--initial-peer-urls=cache-a`.
2. cache-c's syncer seeds its peer list from cache-a via cross-shard LIST — discovers cache-b.
3. cache-c's syncer begins syncing its authoritative shard data (including Cache object copies)
   to both cache-a and cache-b.
4. cache-a's and cache-b's syncers see cache-c's Cache object appear in received shard data
   → discover cache-c → start syncing their own data to cache-c.
5. The mesh is fully connected.

### Authoritative Shard Determination

The syncer watches Shard objects on its source across all shards (wildcard `*`) in the
`system:shard` cluster. A shard is **authoritative** for this syncer if its Shard object
carries `kcp.io/cache` equal to the syncer's own name. Only data from authoritative shards is
synced to peers.

### Watch and Reconciliation Mechanism

The set of resource types available to each shard changes dynamically over time. The syncer
maintains two complementary sets of informers, both driven by CRD discovery on the source.

**Resource type discovery:** The syncer watches CRD objects on the source. Each new CRD
triggers creation of a source informer and a peer informer per known peer for that resource type.
A removed CRD triggers teardown of the corresponding informers.

**Source informers:** One informer per resource type, watching across wildcard `*` shards and
clusters on the source. Each informer uses standard list-watch semantics (resourceVersion,
bookmark events, GONE (410) re-list). Events are filtered client-side to authoritative shards. On
any ADD, UPDATE, or DELETE event, the syncer replicates the change to all known peers.

**Peer informers:** One informer per resource type per peer, watching across wildcard `*` shards
and clusters on that peer. Events are filtered client-side to authoritative shards. On any event,
the syncer compares the peer's object against the source and corrects any divergence:

- Present on peer, absent from source → DELETE from peer.
- Present on peer, differs from source → UPDATE peer to match source.
- Absent from peer, present on source → CREATE on peer.

On startup and on new peer discovery, the peer informer performs its initial LIST, generating
synthetic ADDED events for all existing objects. This serves as the initial reconciliation
automatically — no separate explicit step is needed, and ongoing drift is corrected event-by-event.

**Known scaling limitation:** Total informer count is `M × (1 + P)` where M = resource types and
P = peers. This is a known, accepted limitation to be revisited if scaling constraints are
encountered in practice.

### Write Path to Peers

When syncing an object to a peer, the syncer writes via the peer's Kubernetes API using the
same shard-in-URL mechanism as the shard replication reconcilers:

1. A client is constructed from the peer's `spec.baseURL` with the `--peer-*` credentials.
2. The shard name is injected into the request URL via `WithShardNameFromContextRoundTripper`
   (see `pkg/cache/client/round_tripper.go`), producing the same etcd key structure on the target
   as if the shard had pushed directly.
3. Create, update, and delete operations are issued via the standard Kubernetes API.

Because each shard is authoritative on exactly one cache-server, no write conflicts between
syncers can occur for the same key on a given peer.

### Bootstrap Sequence

1. Connect to the source cache-server using the loopback REST config (embedded mode) or the
   environment kubeconfig (standalone mode).
2. Read the Cache object from source `system:cache:server/system:cache` → derive own cache-server
   name.
3. For each `--initial-peer-urls` entry: cross-shard LIST all Cache objects → seed initial peer
   list; build a client per discovered peer.
4. Start a cross-shard watch of Cache objects on source → ongoing peer discovery; for each newly
   appearing Cache object whose `spec.baseURL` differs from the source, build a client and begin
   syncing to that peer.
5. Start a cross-shard watch of Shard objects on source → maintain the authoritative shard list
   (those carrying `kcp.io/cache == own-name`).
6. Watch CRDs on source → for each resource type, start a source informer and a peer informer per
   known peer; reconcile on events from either side.
7. As new peers are discovered (step 4) or new authoritative shards appear (step 5), start the
   corresponding peer informers; their initial LIST serves as the initial reconciliation
   automatically.

### Deletion Propagation

#### Active Shard Objects

DELETE events from source informers are propagated to each peer immediately, retrying with
backoff until each peer confirms. DELETE events arising from peer informers — object present on
peer but absent from source — are handled by the same retry-until-confirm path.

No periodic re-listing is required: the informer LIST-on-start and GONE (410) re-list semantics
ensure the syncer never permanently loses track of source state after a restart, and peer
informers continuously surface and correct any drift.

Eventual consistency is the correctness guarantee — there is no hard bound on propagation delay.

#### Shard Decommissioning

When a shard is removed, it leaves no authoritative footprint — the syncer stops tracking it
and peer informers filter it out (not authoritative). Stale data for that shard accumulates on all
peers indefinitely. Two options for cleanup are under consideration:

**Option 1: Dedicated cleanup tool (near-term)**
An operator-run tool that, given a shard name, connects to all known peers and deletes all objects
for that shard across all resource types. Explicit, auditable, and does not depend on the syncer
being running.

**Option 2: Autonomous GC on cache-servers (follow-up)**
Cache-servers detect that a shard no longer exists in the mesh — no Shard object in any connected
shard's data — and garbage-collect its data autonomously. Fully automatic and requires no operator
action, but needs changes to the cache-server itself.

Both options can coexist: the dedicated tool as the near-term operational escape hatch, autonomous
GC as the long-term solution.

**Current decision:** Neither is implemented in the initial version. Shard decommissioning is not
yet in scope — the kcp-side workflow for shard removal is not ready.

## Alternatives Considered

**Shard-to-all-peers push from within the shard.** Each shard could push to every cache-server
instead of one. This eliminates the syncer but scales as O(shards × cache-servers), couples
each shard's configuration to the full peer topology, and requires changes to every shard's
`--cache-kubeconfig` handling. Rejected in favour of a dedicated fan-out component.

**Pull-based replication (each cache-server pulls from peers).** Each cache-server would pull
data it needs from peers on demand. Simpler steady-state but requires a request-path dependency on
peer availability and does not produce a local complete copy suitable for serving global list/watch
operations. Rejected.

**Central replication hub.** A single global replication component pushes data from all shards to
all cache-servers. Avoids per-server agents but introduces a single point of failure and a
bottleneck that grows with the product of shards × cache-servers. Rejected.

**Storing cross-cache data in a shared external store.** Cache-servers could share a single etcd
cluster or object store. This removes the replication problem entirely but defeats the purpose of
multiple cache-servers (isolation, geo-distribution, independent scaling) and requires deep
changes to cache-server internals. Rejected.

## Risks and Mitigations

**Informer count growth.** The total informer count `M × (1 + P)` may become large in
installations with many resource types and many peer cache-servers. Mitigated by the accepted
design decision to revisit this if scaling constraints are hit in practice; the initial deployment
target is small peer counts.

**Stale data after shard decommissioning.** Decommissioned shards leave orphan data on all peers
indefinitely until cleaned up. Mitigated short-term by the planned operator cleanup tool; long-term
by autonomous cache-server GC. Operators should run the cleanup tool as part of any shard removal
procedure.

**Split-brain writes if authoritative assignment is ambiguous.** If two cache-servers both believe
they own the same shard (e.g. due to a misconfigured `kcp.io/cache` annotation), both syncers
would push data for that shard to each other, creating a write loop. Mitigated by the invariant
that the `kcp.io/cache` annotation is set by the kcp control plane and must be unique per shard;
the syncer is read-only with respect to this annotation.

**Peer connectivity loss.** If a peer becomes unreachable, the syncer retries writes with
backoff. Data on the unreachable peer will lag; once connectivity is restored the peer informer's
initial re-list and ongoing event stream bring it back to consistency automatically.

**Bootstrap with no initial peer URLs in a new installation.** In a single-cache-server
installation `--initial-peer-urls` is not needed; peers are discovered as they join via Cache
object propagation. In a multi cache-server installation, at least one peer URL must be provided at
startup; subsequent peers are discovered transitively.

**Leader election.** The cache-server currently doesn't employ leader-election, and so
multiple embedded cache-syncer instances may clash. Needs a follow-up.
