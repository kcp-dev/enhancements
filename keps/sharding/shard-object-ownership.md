# Shard Object Ownership by Shards

## Summary

Today every shard self-registers its `Shard` object into the `:root` workspace
on the root shard, over the network, at startup. This proposal moves ownership
of the `Shard` object to the shard itself: the authoritative object lives in
the shard's local `system:shard` logical cluster and is replicated to the
cache server from there. The `:root` workspace only keeps a read-only copy
(a "representation") that the root shard mirrors back from the cache so that
shards remain discoverable in one place.

Tracking issue: [kcp-dev/kcp#4335](https://github.com/kcp-dev/kcp/issues/4335).
Proof of concept: [kcp-dev/kcp#4337](https://github.com/kcp-dev/kcp/pull/4337).

> **Update:** the view and write layer of this KEP (the `:root` mirror and
> the annotation back-sync, option B below) is superseded by the
> [Admin workspace](shard-aggregated-admin-view.md): an aggregated,
> cache-backed view at `/services/admin` with allow-listed writes carried
> through the cache (option C). What remains in scope here is moving the
> authoritative `Shard` object into the shard-local `system:shard` cluster
> (local registration), plus the `Schedulable` condition ack and the shard
> CLI. Rebased onto the Admin workspace, the mirror and back-sync of the POC
> are dropped and root ends up holding no shard state at all.

## Motivation

- A shard cannot finish registration while the root shard is down. With
  shard-owned objects, registration is a local write.
- All shard metadata currently flows through one workspace on one shard. Every
  future field that shards report about themselves (for example `status.used`
  workspace counts from [shard-workspace-limits](shard-workspace-limits.md))
  would add another network call from every shard to root.
- Frequently updated data (capacity, usage, health) is much cheaper as a local
  write plus the existing replication machinery than as a remote call per
  update per shard.

### What we get

1. Shards start and run without the root shard being reachable.
2. One write path for shard data: local object → cache server → consumers.
   Root holds a copy, not the source of truth.
3. Shard configuration (URLs, labels) has one owner: the shard's own
   flags/deployment. An admin edit in root can no longer drift apart from what
   the shard registered.
4. A place to put future shard-reported status (usage counts, heartbeats)
   without extra cross-shard traffic.

### Non-goals

- Rebalancing or moving workspaces (see
  [logical-cluster-migration](logical-cluster-migration.md)).
- General two-way sync between root and shards. This comes up as an option
  below, but is not proposed here.

## Proposal

- Registration writes the `Shard` object into the local `system:shard` logical
  cluster. `system:shard` already exists on every shard and already has the
  Shards API bound into it, so no new bootstrap is needed. Labels come from
  shard configuration (a `--shard-labels` flag) instead of API edits.
- The replication controller replicates `shards` from `system:shard`. Today
  its filter drops everything in `system:*` logical clusters, so this needs a
  carve-out. `Shard` objects in other clusters (created by tests, or left over
  from pre-upgrade shards) keep replicating as before, except representations
  (see below), which are skipped so a shard does not show up twice in the
  cache.
- A new controller on the root shard watches the cache and maintains the
  read-only copies in the `:root` workspace, marked with a
  `core.kcp.io/shard-representation` annotation. It takes over objects left
  behind by pre-upgrade shards, overwrites manual edits, and deletes copies
  whose authoritative object is gone. Objects without the annotation and
  without an authoritative counterpart are left alone, so existing tests and
  mixed-version upgrades keep working without a feature gate.
- Most consumers list shards across all clusters and need no change. A few
  call sites look up a shard by name in the root cluster; these switch to a
  by-name lookup across clusters (shard names are unique). The front-proxy
  keeps watching the `:root` workspace and works unchanged against the
  representations.

The POC in kcp-dev/kcp#4337 implements this end-to-end, including e2e coverage
for the registration → replication → mirror pipeline. It exists to check the
design works, not to lock it in.

### Breaking changes

- The `Shard` object in the `:root` workspace becomes read-only, with the
  exception of a small allow-list of back-synced annotations (see below).
  Other edits (spec, labels, annotations) are overwritten by the mirror.
  Shard configuration moves to the shard's flags/deployment.
- Labels are re-applied from flags on every shard restart. Labels set via the
  API do not survive.

## Open problem: acting on a shard from the control plane

Editing the `Shard` object in root was the one way an admin could act on a
shard centrally. This proposal removes it. Two operations are affected:

1. Cordoning (the `experimental.core.kcp.io/unschedulable` annotation). Today
   only tests set it, but it is the obvious building block for "drain before
   maintenance or migration".
2. Decommissioning. `Shard` objects have no heartbeat. A dead shard's
   authoritative object, its cache copy, and its representation stay around
   until someone deletes them where they live.

Under this proposal, without further machinery both require talking to the
shard's own endpoint (`<shard-base-url>/clusters/system:shard`), or deleting
cache-server state directly if the shard is permanently gone. For cordoning,
option B below is implemented (see Recommendation); decommissioning remains
open.

### Options

| | Option | How | Pros | Cons |
|-|--------|-----|------|------|
| A | Shard-local operations | Document the per-shard endpoints for cordon/decommission | No new machinery; one owner per object | Operating on many shards means talking to each one; no central control point |
| B | Back-sync of selected annotations | The mirror copies an allow-listed set of annotations (e.g. cordon) from the representation back to the authoritative object, using direct shard clients (the workspace scheduler already keeps such a client pool) | `kubectl annotate` in root works again; small change | Root connects to shards again; two writers on one object, even if limited to the allow-list |
| C | Intent objects through the cache | The control plane writes its intent (e.g. a cordon marker) in root; it replicates to the cache; each shard watches the cache and applies the intent to its own object | Central UX without root connecting to shards; uses the replication channel that already exists; also covers decommission ("this shard should go away") | New API surface; eventually consistent; the most design work of the three |
| D | A real API field | `spec.unschedulable` (like on Nodes), owned by shard flag/config only | Single owner, no sync | Cordoning means changing flags and restarting the shard — useless during an incident |

A–C decide how a command reaches the shard; D only decides what the API looks
like. A centrally writable `spec.unschedulable` is not an alternative to B or
C: the write still has to reach the shard-owned object, which is B or C again.
D only stands on its own in the flag-only form above. The end state is
therefore a combination, e.g. C plus a proper spec field.

### Recommendation

Start with B for cordoning, limited to a single allow-listed annotation. The
POC implements it: the mirror treats `experimental.core.kcp.io/unschedulable`
as owned by the representation — an admin sets (or removes) it on the `Shard`
representation in `:root`, the mirror preserves it there instead of
overwriting it, and syncs it back onto the authoritative object over a direct
connection to the owning shard, reusing the logical-cluster-admin credentials
and the client-pool pattern the workspace scheduler already uses for
cross-shard writes. Authorization-wise this means the logical-cluster-admin
identity is allowed into `system:*` logical clusters (previously
system-masters only) and gets get/update/patch on `shards` in the bootstrap
policy.

This accepts B's known cost: root connects to shards again, and there are two
writers on the authoritative object (limited to the allow-list, with the
representation as the single owner of those keys — absence on the
representation means removal).

The receiving shard acknowledges the signal the way a Kubernetes node would:
a controller on every shard keeps a `Schedulable` status condition on its own
authoritative object in sync with the annotation (`False`/`Cordoned` while
cordoned, `True` otherwise). Since the mirror copies status onto the
representation, the condition surfacing in `:root` confirms the cordon was
received and applied end to end. (Kubernetes itself acks `spec.unschedulable`
with the `node.kubernetes.io/unschedulable` taint; shards have no taints, so
a condition is the closest equivalent, and leaves room for future health
reasons the way node conditions do.) When cordon/decommission grow beyond a single
annotation, build C: it is the only option where shards never need root and
root never connects to shards, and the replication channel it needs already
exists. D alone does not work for cordon.

Decommissioning also needs a heartbeat or lease on the authoritative object
(renewed locally, so cheap under this proposal), so that dead shards can be
detected instead of only deleted by hand.
