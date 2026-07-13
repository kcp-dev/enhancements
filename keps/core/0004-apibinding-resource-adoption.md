# APIBinding Resource Adoption Enhancement Proposal

Related: [0005 — APIExport Identity Rotation](0005-apiexport-identity-rotation.md), which
builds on this proposal to remove the "identity reuse is forever" constraint.

## Summary

Deleting an APIBinding today unconditionally deletes every instance of every resource the
binding bound in the consumer workspace. This makes any provider-side API refactoring that
requires consumers to *replace* a binding — most prominently splitting one APIExport into
several, merging exports, or renaming an export — a destructive operation for consumers:
the only supported migration is backup → delete binding → create new bindings → restore,
which loses UIDs, resourceVersions, status, and owner references, and creates an API
outage window.

This proposal adds first-class **resource adoption**: the ability to hand bound resource
instances over from one APIBinding to another (or a set of others) without deleting and
re-creating them, plus a **wait-for-successor deletion policy** as the underlying
building block, preserving the invariant that every instance in storage is covered by a
live APIBinding at all times.
Adoption is zero-copy and therefore requires the successor export to serve the **same
storage** — same APIResourceSchema UID, same identity. This is by design and permanent:
migrations always happen under the same identity. Changing identity is a separate,
subsequent **rotation** operation, defined in KEP 0005.

## Motivation

Consider a provider that ships one APIExport `wildwest` exposing `cowboys.wildwest.dev`
and `sheriffs.wildwest.dev`, and wants to split it into two APIExports `cowboys` and
`sheriffs` (one resource each). The provider side is easy and non-destructive:

* Bound CRDs are keyed by the **APIResourceSchema UID**, so new exports referencing the
  *same* schemas serve a byte-identical API.
* Multiple APIExports may **share one identity** (`spec.identity.secretRef`), so new
  exports reusing the old identity keep the same etcd prefix and the same `identityHash`
  that permission claims and virtual-workspace consumers reference.

The consumer side is where the model breaks down:

1. Two APIBindings cannot both *bind* the same group/resource in one workspace: the
   second binding sits at `BindingUpToDate=False` / `NamingConflicts` until the first
   one goes away. So there is no overlap period.
2. The APIBinding deletion controller (`pkg/reconciler/apis/apibindingdeletion`)
   iterates `status.boundResources` and issues `deletecollection` for every bound
   resource before releasing the `apis.kcp.io/apibinding-finalizer`. There is no way to
   opt out, and it does not consider whether a successor binding is waiting to take the
   resources over.

The result: data that would be perfectly served by the successor bindings — same schema
UID, same identity, same etcd prefix, not a single byte needing to change — is destroyed
purely because the control-plane object that granted access to it was deleted.

### Why adoption is cheap in kcp's storage model

Resource instances are **not** stamped with the binding that bound them. "Ownership" of a
group/resource by a binding exists in exactly two places:

* the `internal.apis.kcp.io/resource-bindings` annotation on the `LogicalCluster` — a
  per-GR lock (`pkg/reconciler/apis/apibinding/logical_cluster_lock.go`) preventing two
  bindings/CRDs from owning the same GR, and
* the binding's own `status.boundResources`.

Instances live in etcd under a prefix determined by the export's **identity hash** and
are served through a bound CRD named by the **APIResourceSchema UID**. Both values are
already recorded per bound resource in
`status.boundResources[].schema.{UID,identityHash}`.

Therefore, handing resources over from binding A to binding B requires **no data
migration whatsoever** when B's export references the same schema (UID) with the same
identity — it is purely a matter of (a) not deleting the instances and (b) transferring
the GR lock. Everything needed to *verify* that safety condition is already in the API.

### Goals

1. Allow a consumer to replace APIBinding X with APIBindings Y (and Z, ...) such that all
   instances of the bound resources survive with unchanged UID, resourceVersion history,
   status, and owner references.
2. Zero data copying: adoption is a control-plane-only handover.
3. Safety by verification: adoption only happens when the successor demonstrably serves
   the same storage (same APIResourceSchema UID, same identityHash). Anything else is
   deletion, as today.
4. Keep today's behavior the default: no change for existing users.

### Non-Goals

* Adoption across **different identities** — permanently out of scope, not just
  deferred. A different identity means a different etcd prefix, and combining a binding
  migration with an identity change makes neither step verifiable. The intended model:
  migrate under the same identity (this KEP), then rotate the identity as a separate,
  subsequent operation ([KEP 0005](0005-apiexport-identity-rotation.md)).
* Adoption across **workspaces** — instances are workspace-local.
* Handover between an APIBinding and a **CRD** (or vice versa) — CRDs use a separate
  etcd prefix by design.
* Schema-divergent handover (successor export referencing a *different* schema for the
  same GR), even if structurally compatible. Same-UID is the alpha criterion; relaxing
  it can be revisited later.
* Any state in which instances exist in storage without a live APIBinding covering
  them. An earlier draft proposed an `Orphan` policy; it was rejected in review (see
  Alternatives) precisely because dangling objects fall outside the resource model.

## Proposal

Two pieces, one building on the other.

### 1. `spec.deletionPolicy` on APIBinding (building block)

```yaml
apiVersion: apis.kcp.io/v1alpha2
kind: APIBinding
spec:
  deletionPolicy: Delete | WaitForSuccessor   # default: Delete
```

* `Delete` — today's behavior, unchanged, remains the default.
* `WaitForSuccessor` — the binding's finalizer is **not released** until every bound
  group/resource has a verified successor (see below) to hand its instances to. Until
  then the binding stays in `Terminating` with
  `BindingResourceDeleteSuccess=False` / reason `WaitingForSuccessor`, listing the
  group/resources still waiting. Instances are never deleted and never invisible:
  throughout the wait they remain served through the deleting binding (read-only,
  per today's deletion semantics), and the moment a successor appears they are handed
  over per-GR.

This preserves a hard invariant the resource model can rely on: **every instance in
storage is covered by a live APIBinding at all times** — either the deleting one or its
successor. There is no dangling state:

* No quota interaction: instances always remain visible and countable through a
  binding, so object-count quota (API-based or etcd-based) never counts objects the
  user cannot see or delete.
* No invisible storage: a migration that stalls is a binding conspicuously stuck in
  `Terminating` with a condition saying exactly what it waits for — first-class,
  debuggable state instead of orphaned bytes.
* Abortable in both directions: patch `deletionPolicy` back to `Delete` to let the
  deletion proceed destructively, or create the missing successor to complete the
  handover.

The field is mutable at any time, including while the binding is terminating, so the
decoupled-timing flow ("delete the old binding now, bring up successors later, e.g.
across a large fleet") still works — the binding simply waits instead of destroying or
stranding anything.

### 2. Adoption: verified per-GR handover

Adoption makes the handover **verified and observable**:

* When the deletion controller processes a binding (any policy), it first computes, for
  each entry in `status.boundResources`, whether a **successor** exists: another
  APIBinding in the same workspace whose referenced export offers the same
  group/resource via a schema with the **same UID** and the **same identityHash**.
  (Successors are typically sitting in `NamingConflicts` waiting for exactly this GR.)
* For every GR with a verified successor, instances are **not deleted** — regardless of
  `deletionPolicy` — and the GR lock is handed to the successor binding atomically in
  the same `resource-bindings` annotation update that releases it, so no third binding
  can race in between. (Equivalently, viewed from the successor's side: when the
  apibinding reconciler evaluates a naming conflict and the conflicting lock holder is
  a deleting binding whose bound resource it can verifiably adopt, it takes the lock
  over instead of reporting `NamingConflicts`.)
* For every GR without a successor, `deletionPolicy` decides: `Delete` deletes (today's
  behavior), `WaitForSuccessor` holds the finalizer and requeues until one appears.
* The successor binding surfaces adoption in status: a new condition
  `ResourcesAdopted` and an event naming the predecessor binding and per-GR instance
  counts. If a would-be successor exists but fails verification (different identity or
  schema UID), it gets a condition explaining *why* adoption did not happen — this is
  the guardrail against the silent-stranding failure mode.

With adoption, the default-policy migration is simply:

```sh
kubectl apply -f apibindings-split.yaml      # pre-create successors
kubectl delete apibinding wildwest           # per-GR handover happens automatically
kubectl wait apibinding cowboys sheriffs --for=condition=Ready
```

> [!IMPORTANT]
> No `deletionPolicy` change is needed on this path — handover applies regardless of
> policy. The policy only decides the fate of GRs that have **no** verified successor at
> deletion time: `Delete` destroys them (today's behavior), `WaitForSuccessor` blocks
> finalization until one appears. Patching to `WaitForSuccessor` before the swap is
> therefore a recommended belt-and-braces step, not a requirement: if a successor turns
> out to be misconfigured (wrong identity, wrong schema) at the moment of deletion, it
> converts the failure mode from data loss into a visible, waiting `Terminating` binding
> that completes the instant the successor is fixed.

The API-unavailability window shrinks to the reconcile gap between lock handover and the
successor binding becoming bound (the bound CRD never goes away — it is keyed by schema
UID and kept alive by the successor's reference).

### Provider-orchestrated migration

A provider running a fleet-wide split does not want to ask every consumer to run the
steps above by hand. The existing permission-claim machinery already supports
orchestrating the whole flow from the provider side — no new API is needed, but the KEP
makes the interaction explicit:

* Without a claim, the APIExport virtual workspace serves `apibindings` **read-only**
  (fallback in `pkg/virtual/apiexport`, filtered to bindings referencing this export).
  Enough to discover consumers, not enough to migrate them.
* With a permission claim on `apibindings.apis.kcp.io` (built-in, empty
  `identityHash`) carrying `create`, `update`/`patch`, and `delete` verbs — accepted by
  the consumer — the provider can, through its own export's virtual workspace:
  pre-create the successor bindings, patch `spec.deletionPolicy` on the old binding,
  and delete it, per consumer workspace.

Two consequences to design around:

* **Access dies with the binding that granted it.** The claim rides on the *old*
  binding; the moment it is deleted, the provider's claimed access through it is gone.
  The successor exports must therefore carry the same `apibindings` claim, and the
  provider should create the successor bindings (and see them accepted) *before*
  deleting the old one — which is the required ordering for adoption anyway, so the
  flows compose rather than conflict.
* **Claim acceptance is spec.** Whoever can write an APIBinding object can also set its
  `permissionClaims[].state: Accepted`. A provider with a write claim on `apibindings`
  can thus create successor bindings with claims pre-accepted. This is the existing
  trust model (the consumer accepted exactly this power when accepting the original
  claim), not something this KEP introduces — but `deletionPolicy` raises the stakes,
  so it is called out: accepting a write claim on `apibindings` means trusting the
  provider with orphan/delete decisions for the data bound through it.

Zero-copy adoption requires the successor export to reuse the predecessor's identity.
Left alone, that makes identity sharing permanent — exports named `cowboys` and
`sheriffs` forever backed by a secret named `wildwest`. KEP 0005 removes that constraint
with provider-driven identity **rotation**: a separate operation, run after the binding
migration has completed, that moves an export (and all its consumers' data) onto a fresh
identity. The sequencing is deliberate and strict — migrate same-identity first, rotate
second, never both at once — so that each step is independently verifiable: the handover
moves no data, and the rotation changes no bindings. This KEP is control-plane-only and
small; 0005 touches the storage layer and can follow independently.

## API Changes

`sdk/apis/apis/v1alpha2`:

```go
// APIBindingSpec
// deletionPolicy controls what happens to instances of bound resources when
// this APIBinding is deleted and no successor binding adopts them.
// "Delete" (default) deletes all instances. "WaitForSuccessor" holds the
// binding's finalizer until every bound group/resource has a verified
// successor binding to adopt its instances; the binding stays in Terminating
// and the instances stay served until the handover completes.
// +kubebuilder:validation:Enum=Delete;WaitForSuccessor
// +kubebuilder:default=Delete
// +optional
DeletionPolicy APIBindingDeletionPolicy `json:"deletionPolicy,omitempty"`
```

New condition types on APIBinding status:

* `ResourcesAdopted` (successor side) — resources were taken over from a deleted
  binding; message lists predecessor and GRs.
* Reason `AdoptionBlocked` on `BindingUpToDate` (successor side) — a predecessor is
  being deleted but this binding does not qualify (identity/schema mismatch), with the
  mismatch spelled out.

`v1alpha1` conversion: the field round-trips through an annotation
(`apis.kcp.io/deletion-policy`), absent meaning `Delete`.

## Implementation Notes (kcp repo)

* `pkg/reconciler/apis/apibindingdeletion`: before `deleteAllCRs`, resolve successors
  per GR (an indexer on APIBindings by referenced export → GR/schema-UID/identityHash
  already exists in spirit via `indexers.APIBindingByBoundResources`-style indexes);
  filter the GVR deletion list accordingly; hand over locks in the same LogicalCluster
  update that would have released them. For `WaitForSuccessor`, unmatched GRs feed the
  existing "resources remaining" requeue machinery instead of being deleted, keeping
  the finalizer held.
* `WaitForSuccessor` must degrade to `Delete` when the **LogicalCluster itself is
  deleting**: workspace teardown deletes bindings as part of removing all content, and
  a binding waiting for a successor that can never come would deadlock the workspace
  in `Terminating`. The deletion controller checks the LogicalCluster's deletion
  timestamp and skips the wait in that case.
* `pkg/reconciler/apis/apibinding`: on conflict evaluation, when the conflicting
  binding has a deletion timestamp, surface `AdoptionBlocked` vs. pending-adoption in
  conditions instead of a bare `NamingConflicts`.
* Bound-CRD lifecycle needs no change: CRDs in `system:bound-crds` are schema-UID-keyed
  and reference-counted across bindings; a successor referencing the same schema keeps
  the CRD alive through the swap.
* Virtual workspaces / permission claims need no change: both key on identityHash,
  which is unchanged by definition of a verified adoption.
* e2e: replicate the wildwest split (one export, two resources → two exports) and
  assert instance UIDs are identical before and after the swap; plus a
  WaitForSuccessor test asserting the binding stays in `Terminating` with the
  `WaitingForSuccessor` condition while no successor exists, that instances remain
  served throughout the wait, and that the handover (and finalization) completes as
  soon as the successor is created.

## Alternatives Considered

* **Backup/restore (status quo).** Loses UIDs, status, ownerReferences; requires
  quiescing controllers; has a hard API-outage window; scales badly with data volume.
* **Manually stripping the `apis.kcp.io/apibinding-finalizer`.** Works by accident for
  the same reason adoption works by design (instances aren't stamped), but races the
  deletion controller, requires privileged access, and bypasses every safety check.
  The existence of this hack is evidence the feature is missing, not a substitute.
* **Provider-side aliasing (an export that "includes" another).** Keeps the old binding
  valid but never lets consumers finish the migration; complexity lands in the export
  model permanently instead of in a one-time handover.
* **Per-object ownership stamping + selective adoption (Kubernetes-style adopt/orphan
  with ownerReferences).** Unnecessary: kcp's storage model already has no per-object
  binding ownership, so stamping would add write amplification only to enable a
  mechanism that works without it.
* **`Orphan` deletion policy (earlier draft of this KEP).** Deletion would release the
  finalizer and leave instances in storage, unreachable until a matching binding
  reappeared. Rejected in review: dangling objects are outside the resource model —
  they occupy etcd while being invisible to (and undeletable by) the user, which
  breaks object-count quota in both directions and invites unknown corner cases in
  anything that assumes every stored object is reachable through the API.
  `WaitForSuccessor` provides the same decoupled-timing migration flow while keeping
  every instance covered by a live binding at all times; a stalled migration is a
  binding visibly stuck in `Terminating` rather than invisible bytes.

## Risks and Mitigations

* **A `WaitForSuccessor` binding waits forever.** Explicit opt-in, visible as a
  `Terminating` binding with the `WaitingForSuccessor` condition listing exactly which
  group/resources lack a successor. Resolvable at any time by creating the successor
  or patching the policy back to `Delete`. Workspace deletion overrides the wait and
  cleans up regardless (workspace teardown deletes all content anyway). Default
  remains `Delete`.
* **Adoption races (successor created concurrently with deletion).** Lock handover and
  lock release happen in a single LogicalCluster annotation update; with
  `WaitForSuccessor`, a successor that misses one evaluation is simply picked up on a
  later requeue — the deletion does not proceed without it.
* **Split-brain across shards.** APIBindings, their LogicalCluster, and the bound
  instances are co-located on the consumer workspace's shard; the handover touches only
  shard-local state. Export/schema lookups go through the cache server as they already
  do for conflict checking.
* **Users expecting adoption across identities.** The `AdoptionBlocked` condition names
  the exact mismatch (identity vs. schema UID) instead of silently deleting or silently
  stranding. The supported answer is: migrate under the same identity, then rotate
  (KEP 0005) — never both at once.
* **Permanent identity sharing.** Inherent to zero-copy adoption; accepted here and
  resolved by KEP 0005 (identity rotation).

> [!IMPORTANT]
> Migrations are always performed under the **same identity**. Identity rotation is a
> separate, subsequent operation (KEP 0005) run *after* the binding migration has
> completed. Migrating APIBindings/APIExports while simultaneously changing identity is
> out of scope by design: never combine the two steps. This keeps each step trivially
> verifiable — the handover moves no data, and the rotation changes no bindings.
* **A provider with a write claim on `apibindings` can stall or delete consumer
  data.** Not a new power in kind — such a claim already allows deleting the binding,
  which today deletes the data. `deletionPolicy` makes the outcome *less* destructive,
  but the trust boundary is documented explicitly in the provider-orchestrated
  migration section: verbs on the claim are the control knob, and consumers who want
  hand-run migrations simply don't accept write verbs on `apibindings`.
