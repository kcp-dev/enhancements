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
re-creating them, plus an **orphan deletion policy** as the underlying building block.
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
* Garbage-collecting orphaned storage. Orphaned instances are bounded by workspace
  lifetime (workspace deletion removes them).

## Proposal

Two pieces, one building on the other.

### 1. `spec.deletionPolicy` on APIBinding (building block)

```yaml
apiVersion: apis.kcp.io/v1alpha2
kind: APIBinding
spec:
  deletionPolicy: Delete | Orphan   # default: Delete
```

* `Delete` — today's behavior, unchanged, remains the default.
* `Orphan` — on binding deletion, the deletion controller **skips instance deletion**
  entirely: it releases the GR locks in `internal.apis.kcp.io/resource-bindings`,
  removes the finalizer, and leaves all instances in storage. They become unreachable
  through the workspace API until some binding binds the same GR with the same identity
  and schema again — at which point they simply reappear, untouched.

The field is mutable at any time before deletion, so the intended flow is "patch policy,
then delete". Deleting a binding with `Orphan` and never creating a successor strands
the data until workspace deletion; the deletion controller emits a warning event
(`OrphanedResources`, with per-GR instance counts) so this is observable, but it is not
prevented — it is the user's explicit choice.

### 2. Adoption: verified per-GR handover

With `Orphan` alone, the migration for the wildwest split becomes:

```sh
kubectl apply -f apibindings-split.yaml      # successors sit in NamingConflicts — expected
kubectl patch apibinding wildwest --type=merge -p '{"spec":{"deletionPolicy":"Orphan"}}'
kubectl delete apibinding wildwest
kubectl wait apibinding cowboys sheriffs --for=condition=Ready
# same objects, same UIDs, same status — nothing was deleted or restored
```

This already works without any per-object operation, but it puts the safety burden on
the user: if the successor exports do *not* reuse the identity or reference different
schemas, the data silently stays stranded instead of reappearing.

Adoption closes that gap by making the handover **verified and observable**:

* When the deletion controller processes a binding (any policy), it first computes, for
  each entry in `status.boundResources`, whether a **successor** exists: another
  APIBinding in the same workspace whose referenced export offers the same
  group/resource via a schema with the **same UID** and the **same identityHash**.
  (Successors are typically sitting in `NamingConflicts` waiting for exactly this GR.)
* For every GR with a verified successor, instances are **not deleted** — regardless of
  `deletionPolicy` — and the GR lock is handed to the successor binding atomically in
  the same `resource-bindings` annotation update that releases it, so no third binding
  can race in between.
* For every GR without a successor, `deletionPolicy` decides: `Delete` deletes (today's
  behavior), `Orphan` orphans with the warning event.
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
> deletion time: `Delete` destroys them (today's behavior), `Orphan` strands them
> recoverably. Patching to `Orphan` before the swap is therefore a recommended
> belt-and-braces step, not a requirement: if a successor turns out to be misconfigured
> (wrong identity, wrong schema) at the moment of deletion, `Orphan` converts the failure
> mode from data loss into a stranded state that reappears once the successor is fixed.

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
// "Delete" (default) deletes all instances. "Orphan" leaves them in storage,
// unreachable until a binding with the same schema and identity binds the
// group/resource again.
// +kubebuilder:validation:Enum=Delete;Orphan
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
  update that would have released them.
* `pkg/reconciler/apis/apibinding`: on conflict evaluation, when the conflicting
  binding has a deletion timestamp, surface `AdoptionBlocked` vs. pending-adoption in
  conditions instead of a bare `NamingConflicts`.
* Bound-CRD lifecycle needs no change: CRDs in `system:bound-crds` are schema-UID-keyed
  and reference-counted across bindings; a successor referencing the same schema keeps
  the CRD alive through the swap.
* Virtual workspaces / permission claims need no change: both key on identityHash,
  which is unchanged by definition of a verified adoption.
* e2e: replicate the wildwest split (one export, two resources → two exports) and
  assert instance UIDs are identical before and after the swap; plus an orphan-without-
  successor test asserting the warning event and that workspace deletion still cleans
  up.

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

## Risks and Mitigations

* **Stranded storage via `Orphan` without successor.** Explicit opt-in, warning event
  with instance counts, bounded by workspace lifetime. Default remains `Delete`.
* **Adoption races (successor created concurrently with deletion).** Lock handover and
  lock release happen in a single LogicalCluster annotation update; a successor that
  misses the handover window finds the lock free and binds normally — instances
  reappear either way, since they were orphaned per the verified-successor rule.
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
* **A provider with a write claim on `apibindings` can orphan or delete consumer
  data.** Not a new power in kind — such a claim already allows deleting the binding,
  which today deletes the data. `deletionPolicy` makes the outcome *less* destructive,
  but the trust boundary is documented explicitly in the provider-orchestrated
  migration section: verbs on the claim are the control knob, and consumers who want
  hand-run migrations simply don't accept write verbs on `apibindings`.
