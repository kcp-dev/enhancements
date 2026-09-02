# The Admin Workspace: Shard Administration Without a Special Root

## Summary

Every shard serves the **Admin 'fake' workspace** at `/services/admin`: an
installation-wide administrative surface, reachable identically through any
shard and exposed in the CLI as the reserved pseudo-workspace `:admin`
(`kubectl ws use :admin`). Its first resource is an aggregated view of all
`Shard` objects, backed by the cache server, writable for exactly one
purpose: allow-listed operational annotations (today: cordoning). Writes are
validated, applied to the cache copy of the target shard, and applied from
the cache to the authoritative object by the shard hosting it — **no
component ever connects to another shard directly**.

Alongside it, the front-proxy discovers shards exclusively through the Admin
workspace of configured **peer shards** (`--shard-peer-kubeconfig`,
repeatable, round-robin with failover), and `Shard` objects everywhere are
protected by admission: shards register themselves and own their objects;
users act through `:admin`.

Tracking issue: [kcp-dev/kcp#4336](https://github.com/kcp-dev/kcp/issues/4336).
This supersedes the interim mirror + back-sync POC of
[shard-object-ownership](shard-object-ownership.md) (kcp-dev/kcp#4337) as the
admin/view layer; see "Relation to shard object ownership" below.

## Motivation

- The front-proxy discovered shards by watching the `root` workspace on the
  root shard: a root outage at proxy start served nothing at all, and a
  runtime outage silently froze routing for new workspaces.
- The `root` workspace was the only place to see and act on shards, making
  root a hard dependency and a special place forever.
- Admin edits of `Shard` objects in root could drift from what shards
  registered (spec is owned by shard flags), and cordoning via direct edits
  had no guardrails.
- Operating shards must not require per-shard kubeconfigs or direct shard
  reachability: many deployments expose only the front-proxy.

### What we get

1. One front door for shard administration (`:admin`), served identically by
   every shard, working through the front-proxy, governed by ordinary RBAC.
2. Shard discovery for the front-proxy without the root shard: any healthy
   peer suffices, new shards appear without peer-list changes, and peer
   failover is exact (one `resourceVersion` space, see below).
3. A single-writer model for `Shard` objects with a guarded, auditable
   exception for operational intent (cordon), delivered without any
   component dialing another shard.
4. Cordon intent survives the target's unavailability: it parks in the cache
   and applies when the hosting shard reconnects. Schedulers react to it
   immediately (they read the cache), before the durable apply.

### Non-goals

- Moving where authoritative `Shard` objects live (that is
  [shard-object-ownership](shard-object-ownership.md); see below).
- General cross-shard aggregation for arbitrary resources.
- Rebalancing/migration of workspaces.

## Proposal (as implemented in POC)

### The Admin workspace virtual workspace

A virtual workspace served at `/services/admin` on every shard
(`pkg/virtual/admin`). Reads (GET/LIST/WATCH) are forwarded to the **cache
server** with shard-wildcard scope, so one request returns every shard's
`Shard` object regardless of which logical cluster it lives in. Cache
bookkeeping annotations are stripped from served objects; objects are
deduplicated by name (a shard-owned `system:shard` copy wins over a legacy
root copy) so mixed-version windows stay correct.

Because every shard proxies the *same* cache server, all shards serve an
identical view in a single `resourceVersion` space. Consumers can fail over
between shards and resume watches with the same RV — the property that makes
peer-based discovery and failover sound. (Consequence: RVs from `:admin` are
cache RVs and must not be used against other endpoints.)

The Admin workspace is the designated home for future installation-wide
admin surfaces, like metrics or other admin operations. 
The authorization model below extends to them automatically.

### Writes: allow-listed intent through the cache

The view is writable for exactly the allow-listed operational annotations
(`MutableAnnotations`, today `experimental.core.kcp.io/unschedulable`).
The update path:

1. The VW validates that *nothing but* the allow-listed annotations changed
   (spec, labels, other annotations all rejected with an ownership error;
   `managedFields` bookkeeping is excluded from the comparison).
2. The validated change is applied to the **cache copy** of the target
   shard, located by the copy's own cluster and shard bookkeeping. Users
   never hold cache credentials; the VW writes with the shard's cache
   client. The cache remains an implementation detail, swappable.
3. The replication controller declares these keys **cache-owned** per GVR
   (`ReplicatedGVR.CacheOwnedAnnotations`): instead of stomping the cache
   copy back to local state, the controller on the shard *hosting* the
   authoritative object applies the cache's values locally (absence meaning
   removal), then normal local→cache sync resumes. All other fields keep the
   strict local→cache ownership.

End-to-end cordon flow:

```
admin → front-proxy → any shard's /services/admin
      → validate (allow-list + SAR) → write cache copy of Shard "shard-3"
      → replication controller on the hosting shard applies it locally
      → schedulers (reading the cache) skip shard-3; ~1s to durable apply
```

### Authorization

Access to a resource in the Admin workspace requires the **same verb on that
resource in the root workspace**, checked via SubjectAccessReview: reading
shards in `:admin` needs read on `shards.core.kcp.io` in root; cordoning
needs update. Permissions stay in one well-known place (RBAC in root, where
kcp-admin already has cluster-admin), deployments can narrow them with
ordinary RBAC, and future Admin workspace resources inherit sane policy.
What can be *written* is further restricted by the storage allow-list,
independent of RBAC.

### CLI: `kubectl ws use :admin`

`:admin` is a reserved pseudo-workspace path (alongside `~`, `-`, `:root`):
it rewrites the kubeconfig server to `<front-proxy>/services/admin`, after
which plain kubectl works (`kubectl get shards`,
`kubectl annotate shard shard-3 experimental.core.kcp.io/unschedulable=true`).
Absolute-path navigation works from inside `:admin`; the current-workspace
display recognizes it. `admin` is reserved as a top-level name at the CLI
layer only — no new cluster-name class, no proxy pseudo-cluster routing; a
raw kubeconfig pointing at `/services/admin` works for any client.

### Front-proxy: peer-based discovery, root demoted to seed

`--shard-peer-kubeconfig` (repeatable): named clusters across the given
kubeconfigs are merged (deduplicated by URL) into a peer list. The proxy
discovers `Shard` objects by informing on the Admin workspace of the peers —
requests distributed round-robin, transport failures move a peer to the back
of the order for a 30s cooldown — **the root workspace is never read for
discovery**. When no peers are configured, the `--root-kubeconfig` server
acts as the sole seed peer (Legacy, will be deprecated) (still consumed via 
*its* Admin workspace), so existing deployments work unchanged. New shards 
are discovered through the Admin workspace view with no peer-list change; 
peers matter only at proxy (re)start. Add stable shards for HA; removing 
decommissioned peers is mandatory.

`--root-kubeconfig` is marked deprecated (flag + field): it remains required
only for APIExport identity resolution until identities are resolvable
without the root shard.

(Also fixed en route: the proxy index permanently lost a shard's routes when
its `baseURL` changed, until proxy restart.)

## Breaking changes

- Direct writes to `Shard` objects are denied for non-system users — cordon
  moves to `:admin`. There is no warn-and-allow window; the error message
  carries the redirection.
- Objects read via `:admin` carry cache-server RVs; they cannot be used for
  writes against other endpoints (and vice versa).
- The proxy warns on `--root-kubeconfig` at startup (deprecation).
