# Workspace Limits for Shards

## Summary

Shards currently accept an unbounded number of workspaces. Performance testing
(presented in the community call on 2025-07-03) showed that a shard that receives
too many workspaces becomes unusable: the garbage collector is constantly
triggered and the shard does not recover without manually deleting workspaces in
etcd.

This proposal adds two per-shard limits on the number of workspaces (logical
clusters) a shard hosts:

- a **soft limit**: when reached, warnings are logged and the workspace scheduler
  prefers other shards,
- a **hard limit**: when reached, no further workspaces are scheduled onto the
  shard.

Both limits are configured per shard via annotations (until we promote to a proper
 API field) on the `Shard` object, are disabled by default, and can be disabled 
 explicitly by setting them to `0`.

Tracking issue: [kcp-dev/kcp#3472](https://github.com/kcp-dev/kcp/issues/3472).

Depends on: [shard-object-ownership](shard-object-ownership.md). That proposal
decides who owns the `Shard` object and where it lives, which changes both
mechanisms this proposal relies on: where the shard publishes its workspace
count, and where the operator sets the limits. See the notes in the proposal
sections below.

## Motivation

A shard has a practical capacity limit determined by its resources (etcd size,
memory, GC behavior). Exceeding it does not degrade gracefully — it bricks the
shard, and recovery requires manual etcd surgery. Operators need:

1. an early warning that a shard is approaching capacity (soft limit), so they
   can scale out before damage occurs. Operators of large deployments automate
   workspace creation and will not notice organic growth without an explicit
   signal;
2. a backstop that prevents a shard from being pushed past its capacity (hard
   limit), forcing action (scale out or raise the limit) instead of silently
   degrading;
3. the ability to disable both limits, e.g. for stress testing to find the real
   capacity of a given setup.

### Goals

1. Soft limit per shard: shards at or over it are deprioritized during workspace
   scheduling, and crossing it is logged.
2. Hard limit per shard: shards at or over it are excluded from workspace
   scheduling; if no shard is available, the workspace stays unscheduled with a
   clear condition and is retried automatically.
3. Limits are configurable per shard by the operator, without restarting the
   shard.
4. A limit set to `0` (or unset) is disabled.
5. Observability: the per-shard workspace count and the configured limits are
   visible to operators (API object and metrics).

### Non-Goals

1. **Rebalancing or draining** of existing workspaces between shards. This was
   discussed on the issue (e.g. using limit values to drain a shard) and is
   deliberately deferred.
2. **Weighted scheduling by workspace "busyness"**. All workspaces count equally;
   a load-aware notion of capacity (similar to pod resource requests) is future
   work.
3. **Resource-based capacity** (CPU, memory, etcd size). The unit of this
   proposal is the workspace count only.
4. **Front-proxy changes.** The issue text frames shard selection as a
   front-proxy concern, but in kcp today the front-proxy plays no role in
   placement: shard selection happens in the `kcp-workspace` controller
   (`pkg/reconciler/tenancy/workspace/`). This proposal changes only that
   controller and adds one small new controller.
5. **Strict enforcement.** The hard limit is best-effort (see Risks); it is a
   guardrail, not a transactional quota.

## Proposal

### API: annotations on `Shard`

Three annotations on the `Shard` object (`core.kcp.io/v1alpha1`), with constants
defined in `sdk/apis/core/v1alpha1/shard_types.go`:

| Annotation | Written by | Meaning |
| --- | --- | --- |
| `experimental.core.kcp.io/workspace-soft-limit` | operator | If the shard's workspace count is at or over this value, the scheduler prefers other shards and warnings are logged. |
| `experimental.core.kcp.io/workspace-hard-limit` | operator | If the shard's workspace count is at or over this value, the shard is excluded from scheduling. |
| `experimental.core.kcp.io/workspace-count` | the shard itself | Observed number of logical clusters currently hosted on the shard. May lag behind the actual count. |

Semantics:

- Values are parsed as integers. A missing, unparseable, or non-positive value
  means the limit is **disabled**.
- A shard is *at* a limit when `limit > 0 && count >= limit`. A hard limit of
  `500` therefore admits at most 500 workspaces.
- There is no cross-field validation. If `hard < soft`, the hard limit simply
  wins. Promotion to validated `spec` fields is future work (see below).

Annotations (rather than new `ShardSpec` fields) keep the API surface small
while the semantics are proven, match the existing
`experimental.core.kcp.io/unschedulable` escape hatch on `Shard`, and follow the
precedent of the `core.kcp.io/max-total-objects` annotation used by the logical
cluster object-count limit. They also survive shard restarts: the shard's
self-registration only updates the spec URL fields of its `Shard` object and
preserves annotations set by others.

Example:

```sh
# in the root workspace
kubectl annotate shard root experimental.core.kcp.io/workspace-soft-limit=450
kubectl annotate shard root experimental.core.kcp.io/workspace-hard-limit=500
```

**Interaction with [shard-object-ownership](shard-object-ownership.md):** that
proposal makes the `Shard` object in the root workspace a read-only copy, so
the operator can no longer set the limit annotations there. The limits then
become shard configuration (flags, like `--shard-labels`), or need the
control-plane-to-shard channel discussed as options B/C in that proposal.
Changing a limit without restarting the shard (goal 3) only survives via that
channel. The two proposals need to be sequenced together.

### Publishing the per-shard workspace count

The scheduling decision for a new workspace is made by the `kcp-workspace`
controller on the shard hosting the **parent** workspace, and it may target any
shard. It lists candidate shards through the cache server's `Shard` informer, so
any data used for the decision must be globally visible.

The count of logical clusters per shard cannot be derived globally today: the
cache server's `LogicalCluster` view is incomplete (only logical clusters that
own `WorkspaceType`s are replicated, see
`pkg/reconciler/tenancy/replicatelogicalcluster/`). However, every shard already
knows its own count precisely — its local `LogicalCluster` informer sees exactly
the logical clusters it hosts (this already drives the
`kcp_logicalcluster_count{shard,phase}` metric).

Therefore each shard publishes its own count onto its own `Shard` object:

- A new, small controller `kcp-shard-capacity`
  (`pkg/reconciler/core/shardcapacity/`) runs on **every** shard (unlike the
  existing root-only `pkg/reconciler/core/shard` stub).
- It watches the shard's local `LogicalCluster` informer. Add/delete events
  enqueue a debounced sync (on the order of 10s) that patches the
  `experimental.core.kcp.io/workspace-count` annotation on the shard's own
  `Shard` object in the root workspace, using the same root-shard client the
  shard's self-registration already uses. The debounce bounds write and
  cache-replication churn on busy shards; a periodic resync self-heals missed
  events. Logical clusters in deletion still count — they consume capacity until
  gone. With [shard-object-ownership](shard-object-ownership.md) in place this
  becomes simpler and cheaper: the count is written to the shard's own local
  `Shard` object instead of over the network to root, and reaches the cache
  server through the normal replication path.
- The `Shard` object is replicated to the cache server, so every shard's
  scheduler sees the count (and the limit annotations) on the same object it
  already lists.
- The controller logs on rising edges — reaching the soft limit (info) and the
  hard limit (warning) — and on falling edges when the count drops below a limit
  again. This satisfies the "print log messages" requirement independently of
  scheduling activity.

### Scheduling changes

All changes are in `chooseShardAndMarkCondition`
(`pkg/reconciler/tenancy/workspace/workspace_reconcile_scheduling.go`):

1. **Hard limit.** In the candidate loop (after the existing
   `experimental.core.kcp.io/unschedulable` check), shards at their hard limit
   are treated as invalid with reason `WorkspaceCapacityExhausted`. This feeds
   the existing aggregation: if no valid shard remains, the workspace gets
   `WorkspaceScheduled=False` with reason `Unschedulable`, and the existing
   retry machinery re-enqueues unschedulable workspaces whenever any `Shard`
   object changes — which includes the count-annotation updates made by
   `kcp-shard-capacity`. A shard dropping below its hard limit therefore
   automatically unblocks pending workspaces, exactly like removing the
   `unschedulable` annotation does today.
2. **Soft limit.** The final uniform-random pick over valid shards becomes a
   tiered pick: valid shards under their soft limit form the preferred tier and
   the target is chosen uniformly at random within it. Only if the preferred
   tier is empty does the scheduler fall back to shards at/over their soft limit
   (but under their hard limit), logging a warning that all candidates are above
   their soft limit.
3. **No re-check after placement.** The scheduler is a two-phase commit: phase
   one records the chosen shard on the `Workspace`, phase two creates the
   `LogicalCluster` on that shard. Capacity is only checked during *selection*
   (phase one). Re-checking at the phase-two revalidation would strand a
   workspace forever if the shard filled up in between; completing an in-flight
   placement on a just-filled shard is the lesser evil.

`Workspace.spec.location.selector` continues to work unchanged; limits are
applied to the shards matching the selector.

### Observability

- `experimental.core.kcp.io/workspace-count` on each `Shard` object (human- and
  automation-readable).
- New gauge `kcp_shard_workspace_limit{shard, type="soft"|"hard"}`, published
  only for enabled limits (bounded cardinality, following the object-count limit
  metrics). Combined with the existing
  `kcp_logicalcluster_count{shard,phase}` gauge this makes
  "count vs. limit per shard" a one-expression alert.
- Log lines on soft/hard limit crossings (rising and falling edges) from
  `kcp-shard-capacity`, and a warning from the scheduler whenever it has to
  place a workspace above the soft limit.

## Risks and Mitigations

- **The hard limit is best-effort.** The count is debounced, replicated through
  the cache server with some lag, and multiple parent shards schedule
  concurrently against the same targets. The overshoot is bounded by roughly
  the workspace creation rate times the staleness window. This is the same
  class of race the existing `unschedulable` annotation has (documented in
  `test/e2e/reconciler/workspace/controller_test.go`) and is acceptable for a
  guardrail: operators should set the hard limit with headroom below the actual
  cliff. If needed, the debounce can be shortened, or the publisher can switch
  to immediate writes when close to the hard limit.
- **Typos silently disable a limit.** Annotations are not validated by the API
  server. Mitigated by the limit metrics and crossing logs (an operator who sets
  a limit can verify it took effect); properly solved by future promotion to
  validated spec fields.
- **System logical clusters count too.** The root shard hosts system logical
  clusters (root workspace etc.); they count toward its limits. This is
  documented rather than special-cased — the capacity concern applies to every
  logical cluster equally.
- **A fully limited fleet blocks workspace creation.** If all shards are at
  their hard limit, new workspaces remain unscheduled (with a clear condition)
  until an operator adds a shard or raises a limit. This is the intended
  behavior — it replaces silent shard destruction with an actionable signal —
  but must be prominent in documentation.

## Alternatives

1. **`Shard.spec.capacity` fields (+ optional server flags).** A proper
   validated API (`softWorkspaceLimit`/`hardWorkspaceLimit` with CEL validation)
   was considered and remains the likely end state, but starts with a bigger API
   commitment. Annotations let the semantics be proven first and can be promoted
   with a migration later.
2. **Reusing the dormant `ShardStatus.Capacity` field.** `ShardStatus` already
   contains an unused `Capacity corev1.ResourceList`. Its name and comment imply
   *limit* semantics, not usage, and `resource.Quantity` comparisons are
   awkward for a simple count. Left untouched.
3. **Enforcement in admission.** An admission plugin (like the
   `core.kcp.io/LogicalClusterObjectCountLimit` plugin) cannot enforce a
   per-target-shard limit: `Workspace` CREATE admission runs on the shard of
   the *parent* workspace, before a target shard has been chosen.
4. **Soft limit only** (issue alternative 1). Insufficient: without a hard
   backstop, sustained creation pressure still bricks the shard.
5. **Preventing resource exhaustion altogether** (issue alternative 2). The
   better long-term answer, but far larger in scope; limits are the pragmatic
   guardrail and remain useful even then (mis-set limits notwithstanding).

## Future Work

- Promote the annotations to `Shard.spec` fields with CEL validation
  (`hard >= soft`), plus server flags to seed them at shard self-registration.
- A `ShardCapacityAvailable` condition on `Shard`.
- Rebalancing/draining of workspaces between shards, including the
  limit-value-driven drain semantics sketched in the issue discussion.
- Load-aware ("busyness") capacity instead of a plain count.
