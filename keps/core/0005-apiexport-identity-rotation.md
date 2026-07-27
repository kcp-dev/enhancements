# APIExport Identity Rotation Enhancement Proposal

Related: [0004 — APIBinding Resource Adoption](0004-apibinding-resource-adoption.md).
The two proposals are strictly sequenced: binding migrations (0004) always happen under
the **same identity**; rotation (this proposal) is a separate, subsequent operation on an
already-settled export. Combining a binding migration with an identity change is out of
scope of both proposals by design — each step stays independently verifiable (the
handover moves no data, the rotation changes no bindings).

## Summary

An APIExport's identity is immutable and determines the etcd prefix under which all bound
resource instances are stored. Two consequences compound over a system's lifetime:

1. **Zero-copy adoption (KEP 0004) requires identity reuse.** After splitting export
   `wildwest` into `cowboys` and `sheriffs`, both new exports are forever backed by a
   secret named `wildwest` in the provider's `kcp-system` namespace. Old-enough systems
   accumulate stale identities, secrets named after long-deleted exports, and N exports
   ambiguously sharing one identity — with no way to converge back to clean per-export
   identities.
2. **There is no rotation story at all.** The identity secret gates wildcard and
   virtual-workspace access; if it leaks, there is today no supported way to move an
   export (and its consumers' data) onto a fresh identity.

This proposal adds an **identity lifecycle**: a storage-level identity migrator,
provider-driven **identity rotation** on the APIExport, and observability for identity
sharing (`IdentityShared`). Identity reuse becomes a migration-time tool that can be
undone, not accumulated debt: migrate bindings under the same identity (KEP 0004), then
rotate the export onto a fresh identity as a follow-up operation.

## Motivation

### Storage mechanics: why identity migration is tractable

Bound instances are stored under

```
/registry/<group>/<resource>/<identityHash>/<cluster>/[<namespace>/]<name>
```

The identity hash is injected as a path segment by the RESTOptionsGetter in kcp's
apiextensions fork (`rest_options_getter_kcp.go`), taken from the `apis.kcp.io/identity`
annotation on the bound CRD. Migrating a workspace's data from identity A to identity B
is therefore a prefix-to-prefix copy: object bytes — including UID, status,
ownerReferences — are preserved verbatim; only resourceVersions change (new etcd
revisions), which is Kubernetes-legal and merely restarts watches.

There is existing precedent for tracking exactly this kind of drain:
`status.boundResources[].storageVersions` records every version ever persisted so a
migration controller can drain and then prune it. Identity gets the same treatment:

```yaml
status:
  boundResources:
  - group: wildwest.dev
    resource: cowboys
    schema: {name: v1.cowboys.wildwest.dev, UID: ..., identityHash: <B>}  # exists today
    identityHashes: [<A>, <B>]   # NEW field (this KEP): every identity under which
                                 # instances may exist; the migrator drains A -> B,
                                 # then removes A
```

(`schema.identityHash` — singular — exists today on `BoundAPIResourceSchema`;
`identityHashes` on `BoundAPIResource` is introduced by this proposal.)

The list itself carries no direction — it is an unordered set. The drain **target** is
always the current `schema.identityHash`, which the rotation controller updates to the
new identity when the rotation starts; every *other* entry in `identityHashes` is a
drain **source**. This mirrors how `storageVersions` works: the current version comes
from the schema, the list only records what may still exist in storage. A drain is
complete when `identityHashes` has shrunk back to exactly `[schema.identityHash]`.

### Goals

1. Allow a provider to **rotate** an APIExport's identity in a controlled, tracked
   procedure that migrates every consumer workspace — preserving UIDs, status, and
   ownerReferences — and ends with the old identity secret deletable.
2. Make identity sharing **observable**, so operators can find rotation candidates
   instead of discovering shared identities archaeologically.
3. No new steady-state cost: after a completed rotation, exactly one identity per
   bound resource remains, with no duplicate data and no dual-prefix serving.

### Non-Goals

* **Cross-identity adoption** — combining a binding migration (KEP 0004) with an
  identity change. Binding migrations always happen under the same identity; rotation
  operates only on an export whose bindings are settled. Never both at once: this keeps
  each step independently verifiable (the handover moves no data, the rotation changes
  no bindings).
* Changing the meaning of identity. The hash remains the storage key and the token that
  permission claims and wildcard consumers reference; rotation *changes* the hash, it
  does not decouple anything from it.
* Automatic rotation. Rotation is always operator-initiated on the provider side.
* Cross-workspace or cross-shard data movement beyond what the per-workspace prefix
  copy requires — instances stay in their workspace, on their shard.
* Migrating between CRD storage and APIBinding storage.

## Proposal

One storage-level building block, one flow on top of it, one observability aid.

### Building block: the identity migrator

A per-shard controller that acts on any bound resource whose `identityHashes` contains
more than one entry. The drain target is the current `schema.identityHash`; every other
entry is a source (the set encodes no direction — see above). For each affected
workspace it:

1. **fences the workspace** by reusing the existing maintenance mechanism: the
   `core.kcp.io/inactive` annotation on the `LogicalCluster` (phase `Inactive`,
   introduced for workspace migration). It cancels active connections and rejects
   requests — exactly what the copy needs, since in-flight writes and watches must not
   straddle the flip. This is coarser than strictly necessary (the whole workspace is
   offline, not just the affected group/resource), but it is battle-tested and already
   understood by clients. A finer, per-GR fence is deliberately left as a follow-up
   (see "Follow-up: per-GR write-only fence") rather than a requirement,
2. copies all keys from `/registry/<g>/<r>/<oldHash>/<cluster>/...` to
   `/registry/<g>/<r>/<newHash>/<cluster>/...` via storage handles — never through the
   workspace API, which is what preserves UIDs and bytes,
3. verifies object counts between the prefixes,
4. flips serving to the new prefix (the bound CRD's `apis.kcp.io/identity` annotation)
   and removes the `core.kcp.io/inactive` annotation (workspace returns to `Ready`),
5. deletes the old prefix and prunes the old hash from `identityHashes`.

Because `identityHashes` keeps both hashes until step 5 completes, a crashed migrator
resumes idempotently; count verification gates old-prefix deletion.

### The flow: provider-driven identity rotation on the APIExport

`spec.identity` on the APIExport stays **immutable for users** — rotation is not a spec
edit. It is requested through a dedicated, one-shot resource, following the same pattern
as Kubernetes' `StorageVersionMigration` (which the migrator itself is modeled on):

```yaml
apiVersion: apis.kcp.io/v1alpha2
kind: APIExportIdentityRotation
metadata:
  name: cowboys-rotate-2026-07        # in the provider workspace, next to the APIExport
spec:
  export: cowboys                     # APIExport in this workspace
  newIdentity:
    secretRef: {namespace: kcp-system, name: cowboys}   # fresh secret, pre-created
  aliasRetirement: Manual             # Manual | After (duration) | Immediate (revoke)
status:
  phase: Pending | Migrating | AliasActive | Completed | Failed
  oldIdentityHash: <A>
  newIdentityHash: <B>
  migratedBindings: 37
  totalBindings: 120
  conditions: [...]
```

A dedicated object rather than a mutable spec field because:

* **Options need a home.** Alias retirement policy (`Manual`/`After`/`Immediate`) is
  part of the request, not of the export's steady-state identity.
* **Audit trail.** Completed rotation objects are a durable record of when identity
  changed and why claims referencing an old hash exist; a spec edit leaves no trace.
* **RBAC separation.** Granting someone rotation rights is distinct from granting
  export-spec edit rights.
* **Guarded transitions.** Admission on this resource is where the invariants live: at
  most one active rotation per export, and creation is rejected while any binding of
  the export is still mid-migration (KEP 0004 handover in progress) — never both at
  once.
* **No accidental fleet migrations.** A `kubectl patch` on `spec.identity` cannot
  trigger a storage migration across every consumer workspace; today's immutability
  stays exactly as is, and only the rotation controller updates `spec.identity` (as a
  system actor, when the rotation completes).

#### Alias retirement policies

Once the data drain finishes, the old hash no longer locates any data — it survives
purely as an **equivalence token** so that cross-export permission claims (and wildcard
consumers) still referencing it keep working (see the claims section). "Retirement"
removes that equivalence: the old hash stops resolving, and anything still referencing
it becomes invalid — the same state as referencing an identity that never existed.
Retirement is therefore purely an access/visibility event, never a data event; a stale
claimant fixes it by updating to the new hash, with nothing lost in between.

`spec.aliasRetirement` decides when that happens:

* **`Manual`** (default) — the alias lives until the provider explicitly retires it.
  The rotation object stays in `AliasActive`, and the provider watches the
  `ClaimIdentityRotated` conditions across claiming exports; when the count of stale
  claimants reaches zero (or the provider decides the stragglers' breakage is
  acceptable), they retire the alias by updating the rotation object. Safest, and the
  right choice when you don't control the claimants.
* **`After: <duration>`** — a deprecation window with an announced deadline. The timer
  starts when the rotation enters `AliasActive`; claimants see the deadline in the
  `ClaimIdentityRotated` condition message. Use when there are many third-party
  claimants and the provider wants bounded convergence instead of waiting on the last
  straggler forever.
* **`Immediate`** — no alias window at all: the old hash stops resolving the moment the
  drain completes. Every stale cross-export claim breaks at once — which is the point:
  this is the leaked-secret remediation mode, where anything still trusting the old
  identity *should* break. Never the default.

The field is mutable while the rotation is in `AliasActive`, but only toward earlier
retirement (`Manual` → `After` → `Immediate`): a provider can always shorten the
window or end it now. Retirement itself is irreversible — once the alias is gone,
re-establishing the old hash is not supported (it would amount to un-rotating; if that
is truly needed, it is a new rotation in the opposite direction).

The procedure the rotation object drives:

1. `Pending` → controller validates the new secret, computes the new hash, publishes
   the old hash as an alias in the export status.
2. `Migrating` → fans out per-workspace drains across all bindings of the export
   (existing binding-by-export index), each one exactly the migrator flow above;
   progress in `migratedBindings/totalBindings`.
3. `AliasActive` → all data drained; the old hash lives on only as a claim alias until
   retirement per `spec.aliasRetirement`.
4. `Completed` → alias retired, old identity dead, its secret deletable.

Aborting: deleting the rotation object while `Pending` is a no-op; while `Migrating` it
reverses direction — the migrator's target is always the current `schema.identityHash`,
so flipping that back turns already-migrated workspaces into ordinary drain sources.
Each workspace is fully on one identity or the other at all times, so there is no
partially-rotated state to get stuck in.

For the wildwest split this is the convergence path, always in this order: reuse the
identity for the zero-copy split (KEP 0004), wait until all consumers have migrated and
the bindings are settled, then rotate `cowboys` and `sheriffs` onto fresh identities at
leisure, and finally delete the stale `wildwest` secret.

### Authorization: rotation is a bindable platform capability, not an export-owner power

The disruptive primitive in a rotation is not the data copy — it is **step 1 of the
migrator: fencing consumer LogicalClusters** by setting `core.kcp.io/inactive` on them.
That fence takes an entire consumer workspace offline (reads included, all resources, not
just the bound group/resource — see the migrator flow). An actor able to trigger it
across every binding of an export can black out every consumer of that export at will,
and loop it. This is a consumer-facing DoS vector, so the ability to rotate must be a
capability the platform **grants explicitly**, not an implicit power of owning an
APIExport.

The requirement is that a provider-workspace **cluster-admin cannot self-authorize**
rotation. That rules out gating on plain in-workspace RBAC (a local cluster-admin grants
themselves any local verb). It also, after discussion, rules out the two obvious
"authorize against root" designs — both are recorded under Alternatives Considered as
rejected:

* a dedicated `rotate` **verb** checked via SAR against root — RBAC granting that verb
  has to live somewhere, and special-casing a verb means `*` in a role no longer cleanly
  means `*`; and
* the same object **plus verb** created directly in the root workspace — reintroduces
  "root is magic, the platform does special things there," the exact coupling
  `LogicalClusterMigration` already suffers and which we want to stop growing.

**The chosen model: rotation is served by a platform-owned APIExport that you bind to
use.** `APIExportIdentityRotation` is not a built-in type available everywhere; it is a
resource exported by a platform-owned **maintenance APIExport** (working name
`maintenance.kcp.io`) living in the **root** workspace — the same place, and the same
pattern, as the `tenancy` APIExport. `LogicalClusterMigration` has the identical
"platform-owner button with nowhere to live" problem and is the natural co-tenant of the
same export, so this is one home for privileged maintenance surface rather than a
one-off.

Authorization then falls out of the existing binding machinery, with no new verb and no
special-casing of root in the authorizer:

* To obtain the `APIExportIdentityRotation` type in a workspace, a provider must **bind**
  `maintenance.kcp.io`. Binding an APIExport already requires the `bind` permission on
  that *specific* APIExport, evaluated **in the export's own workspace (root)** — not in
  the binder's workspace. A provider-workspace cluster-admin has no authority in root, so
  they cannot grant themselves `bind` on a root-owned export: the self-authorization door
  is closed by the ordinary bind check, not by a bespoke rule.
* Once bound, `*` still means `*` in the workspace where the binding lives — normal RBAC
  semantics, no confusing verb carve-out. The provider creates the rotation object next
  to their APIExport as before; the capability rode in on the binding.
* **Delegation** is just "grant `bind` on `maintenance.kcp.io` in root" to a trusted
  provider — explicit, auditable, revocable, and expressed entirely with existing RBAC on
  a concrete object. A delegated provider can then rotate (and, in principle, loop it);
  that residual capability is intended and bounded by the per-export cooldown and
  one-active-rotation-per-export invariants below, not a hole.

This keeps the whole thing inside mechanisms kcp already has (APIExport, bind
authorization, maximal permission policy) instead of inventing a platform-only verb or
enshrining root as a magic execution site. The one platform-owned piece is *who may bind
the maintenance export*, which is exactly the knob a platform owner should hold.

> [!IMPORTANT]
> This holds only while the rotation controller is the sole path to (a) setting
> `core.kcp.io/inactive` on a not-owned LogicalCluster and (b) the storage drain. By
> kcp's workspace-isolation model there is no other path (the controller is a system
> actor; a provider-workspace admin cannot write a foreign LogicalCluster directly); this
> KEP states that as an explicit security assumption so any future API that could fence a
> foreign workspace is evaluated against it.

Defense-in-depth invariants (enforced by admission on `APIExportIdentityRotation`):

* At most one active rotation per export (already required for correctness).
* A **minimum interval between completed rotations** of the same export, so even a
  delegated-but-compromised provider cannot loop rotations into a sustained fence.

### Interaction with cross-export permission claims

Identity is not only a storage key — it is the public token by which **other exports
claim resources**. A third-party export claims `cowboys.wildwest.dev` by writing the
producer's `identityHash` into its own `spec.permissionClaims[]`, and consumers record
the same hash when accepting the claim on their bindings. Neither object is owned by
the rotating provider, and both may live on **other shards**.

Rotation must therefore not assume it can atomically rewrite every reference. The
mechanics that make this dangerous (from `sdk/apis/apis/v1alpha2/permissionclaims`):

* Claimed objects are labeled with key `claims.internal.apis.kcp.io/<hash(claiming
  export)>` and **value `hash(claim)` — where the claim includes the identityHash**.
* The claiming export's virtual workspace filter hashes the *export's* claim spec,
  while the consumer-side labeler hashes the *binding's* accepted claim. If the two
  ever hash different identity references, claimed objects silently disappear from the
  claiming export's view (the exact divergence class of kcp issue #4198).

So a naive rotation — flip the hash, tell everyone to update — creates a window where
every cross-export claim on the rotated identity goes dark. Instead:

**Alias with canonical-hash normalization.** During rotation (and for a deprecation
window after), the rotating export publishes both hashes: the export's status carries
the old hash as an **alias** of the new canonical one. Export status is replicated
through the cache server, so every shard can resolve the alias (the existing
`APIExportByIdentity` index is extended to also index alias hashes). Both places that
hash claims — the virtual-workspace filter builder and the permissionclaim labeler —
**normalize any alias hash to the canonical hash before hashing**. A claim still
referencing the old identity and a claim already updated to the new one then produce
identical label keys/values: nothing goes dark, and claim owners can update their specs
lazily.

Lifecycle of the references:

1. During the window, claiming exports whose claims reference an alias hash get an
   informational condition (`ClaimIdentityRotated`, naming the canonical hash);
   consumers see the equivalent on their bindings. Everything keeps working.
2. Owners update specs / re-accept at their own pace; the existing permissionclaimlabel
   controller re-labels on claim change as it does today — no new labeling machinery.
3. When the provider ends the window (explicitly, or after a configured default), the
   alias is removed from export status. Claims still referencing the old hash stop
   resolving and surface as invalid — the same state as referencing any nonexistent
   identity today.

For **leaked-secret remediation** the provider can skip the deprecation window
(immediate revoke): stale claims break at once, which is the point — the old identity
must stop being honored. This is an explicit flag on the rotation request, not the
default.

Cross-shard, nothing new is needed beyond the alias replication: claim resolution
already goes through cache-server-fed informers, and the labelers run shard-locally in
the workspaces where the claimed objects live.

### Follow-up: per-GR write-only fence

The alpha reuses the coarse `core.kcp.io/inactive` fence, which takes the whole consumer
workspace offline for the duration of the copy even though rotation only ever touches one
group/resource's etcd prefix. A natural improvement — explicitly a **follow-up, not part
of this KEP** — is a per-GR, write-only fence: pause mutating requests to just the
rotating GR and leave reads and every other resource in the workspace serving normally.

This is feasible on the existing machinery without new architecture. The fence lives in
the same HTTP filter chain as today's whole-cluster fences
(`pkg/server/filters/inactivelogicalcluster.go`,
`pkg/server/filters/migratinglogicalcluster.go`), which already carry the request's
`RequestInfo` — the migrating filter branches on `requestInfo.Verb` today, and the same
`RequestInfo` also carries `APIGroup`/`Resource`. A per-GR fence would branch on those to
reject only mutating verbs on the rotating GR, driven off the in-progress
`identityHashes` state this KEP already introduces (a GR is fenced iff its
`identityHashes` has more than one entry) rather than a workspace-wide boolean. Same-shard
rotation also sidesteps the reason the migrating filter has to special-case list/watch
resource versions (that concern is cross-shard).

It is intentionally deferred, not adopted now, because **narrowing the fence to a single
GR does not obviously bound the blast radius to that GR.** Objects of other resources may
be indirectly coupled to the rotating one — owner references, controllers that watch the
rotating GR and write elsewhere, admission/validation that reads across resources, quota
— and letting those keep mutating while the target GR's storage is mid-flip could produce
exactly the straddling inconsistency the coarse fence was chosen to avoid. We do not yet
know the full set of such couplings, so the responsible default is the whole-workspace
fence (correct, if blunt) for alpha, and to gather feedback on real rotations before
scoping the fence down. This follow-up therefore strictly *improves* the outage story
without changing the alpha's correctness guarantees.

### Observability: `IdentityShared`

The apiexport reconciler gains an informational condition when an export's identity
secret is referenced by more than one export (`IdentityShared`, listing the co-owners),
so operators can find rotation candidates as a queryable signal.

### Costs to be explicit about

* **Permission claims and wildcard consumers reference the identityHash.** Rotation
  changes the hash. The alias mechanism (see the cross-export claims section) keeps
  existing references working through the deprecation window, so updates are lazy, not
  a flag-day — but they still must happen: claim owners update their export specs,
  consumers re-accept on their bindings, and wildcard clients re-resolve before the
  alias is retired. This is inherent — the hash *is* the identity — and is the reason
  rotation is a deliberate, tracked procedure rather than a secret edit.
* **A brief per-workspace outage** during the copy: the fence reuses the
  `core.kcp.io/inactive` maintenance annotation, which takes the whole workspace
  offline (reads included) and cancels in-flight connections — the same behavior
  clients already tolerate during workspace migration between shards. The window is
  bounded by the size of the affected resources in that one workspace.
* **resourceVersions change.** Controllers see a watch restart / relist, the same as
  any storage migration.

## API Changes

`sdk/apis/apis/v1alpha2`:

```go
// BoundAPIResource
// identityHashes is a NEW field introduced by this proposal (today only the
// singular schema.identityHash exists, on BoundAPIResourceSchema). It lists
// every identity hash under which instances of this resource may exist in
// storage. Mirrors storageVersions: a migration controller drains old
// identities and prunes them from this list. Normally it contains exactly
// the schema's current identityHash.
// +optional
// +listType=set
IdentityHashes []string `json:"identityHashes,omitempty"`
```

No APIBinding *spec* changes: rotation is entirely provider-driven, and consumers never
request an identity change — their bindings are migrated in place and only observe
progress via status.

New resource `APIExportIdentityRotation` (apis.kcp.io/v1alpha2, see the flow section
for the full shape): one-shot rotation request living next to the APIExport, with
`spec.{export, newIdentity.secretRef, aliasRetirement}` and
`status.{phase, oldIdentityHash, newIdentityHash, migratedBindings, totalBindings}`.
Admission on it enforces the invariants: one active rotation per export, rejection
while any binding of the export is mid-migration (KEP 0004), and a minimum interval
between completed rotations of the same export. The type itself is **not** built-in
everywhere: it is served by a platform-owned maintenance APIExport (working name
`maintenance.kcp.io`) in the root workspace, so possessing the type at all requires
binding that export — see Authorization. That binding, gated by the ordinary `bind`
permission on the export (checked in root), is the trust-boundary control: rotation is
*not* implied by owning the APIExport being rotated and cannot be self-authorized by a
provider-workspace cluster-admin. Platforms delegate the capability simply by granting
`bind` on `maintenance.kcp.io`.

APIExport:

* `spec.identity` stays **immutable for users**, exactly as today. Only the rotation
  controller (system actor) updates it, when a rotation completes.
* `status` additionally publishes alias hashes while a rotation's alias window is
  active (consumed by the claim-hash normalization and the `APIExportByIdentity`
  index).
* New informational conditions: `IdentityShared` (identity secret referenced by
  multiple exports, listing co-owners) and `IdentityRotationInProgress` (pointing at
  the active rotation object).

New condition on APIBinding status:

* `IdentityMigrationCompleted` — False while a rotation drain is in progress for any
  bound resource of this binding, True when all old prefixes are verified empty and
  pruned.

`v1alpha1` conversion: `identityHashes` is status-only and simply dropped on
down-conversion; v1alpha1 clients see the current identityHash as today.

## Implementation Notes (kcp repo)

* The migrator lives with the other storage-touching machinery: a per-shard controller
  (pattern: the storage-version migration path that `storageVersions` was designed
  for), driven by `identityHashes` entries with more than one element. The prefix
  injection point is the RESTOptionsGetter in the apiextensions fork
  (`rest_options_getter_kcp.go`); the migrator reads via a storage handle for the old
  hash and writes via one for the new hash.
* Rotation is a thin orchestration over the same migrator: enumerate bindings by export
  (existing index), fan out per-workspace drains, report progress on the export.
* The fence is the existing `core.kcp.io/inactive` LogicalCluster annotation
  (`LogicalClusterPhaseInactive`, added for workspace migration): connection
  cancellation and request rejection come for free, and no new fencing mechanism is
  introduced. The migrator sets it before the copy and removes it at the flip.
* e2e: a rotation test asserting UIDs survive, the old prefix is drained, duplicates
  never appear, and the old identity secret becomes deletable; a crash-resume test
  killing the migrator mid-drain; a negative test asserting rotation is rejected while
  bindings of the export are still mid-migration (KEP 0004 handover in progress); a
  cross-export claims test asserting a third-party export claiming the rotated
  identity keeps seeing its claimed objects throughout the rotation (alias
  normalization), gets `ClaimIdentityRotated`, and loses access only after the alias
  is retired.

## Alternatives Considered

* **Accept permanent identity sharing (do nothing).** Works functionally, but identity
  secrets become unauditable over time, secret leakage has no remediation, and the
  provider's `kcp-system` fills with secrets whose names no longer correspond to any
  export.
* **API-level migration (read old, create new through the workspace API).** Loses UIDs
  and status exactly like backup/restore; the entire point of storage-level migration
  is byte preservation.
* **Dual-prefix serving (serve old and new prefixes simultaneously during migration).**
  Avoids the write fence but creates name-collision ambiguity between prefixes and
  doubles list/watch fan-in; a short fence is simpler and bounded.
* **Decoupling storage location from identity (indirection table).** Would make
  rotation free but adds a lookup to every request path and a new global table to
  shard; rejected for complexity disproportionate to the frequency of rotation.
* **Authorizing rotation via a dedicated `rotate` verb checked against root.** Gate
  `create` on `APIExportIdentityRotation` on a new verb, SAR'd against the root
  workspace. Rejected: the RBAC granting that verb still has to live somewhere, and
  special-casing a single verb erodes the meaning of `*` in roles (a wildcard role would
  or wouldn't include `rotate` depending on bespoke rules) — confusing and easy to get
  wrong. It also still leans on "authorize against root" as a one-off in the authorizer.
* **A rotation object living directly in the root workspace (object + verb in root).**
  Keeps `*` meaning `*` by putting the privileged object where the platform owner
  already has authority, but reintroduces "root is special and we do magic there" — the
  same coupling `LogicalClusterMigration` already has and that we want to stop growing.
  Rejected in favor of the bindable maintenance APIExport, which needs no new verb and no
  root-only execution site: the only platform-held knob is who may `bind` the export.

## Risks and Mitigations

* **Rotation as a consumer-facing DoS vector.** The fence (`core.kcp.io/inactive`) takes
  whole consumer workspaces offline, so an actor able to loop rotations across an
  export's bindings could black out every consumer. Mitigated by making rotation a
  platform-owned capability rather than an implicit power of export ownership (see
  Authorization): the `APIExportIdentityRotation` type is served by a platform-owned
  maintenance APIExport in root, so wielding it requires binding that export, which is
  gated by the ordinary `bind` permission checked *in root* — a permission a
  provider-workspace cluster-admin cannot self-grant. Per-export cooldown plus
  one-active-rotation-per-export bound even a delegated provider. The rotation controller
  being the sole path to fencing a not-owned LogicalCluster is stated as an explicit
  security assumption.
* **Migration interrupted mid-drain.** `identityHashes` keeps both hashes until the
  drain is verified complete, so a crashed migrator resumes idempotently, and the
  inactive fence means no writes land in-between. Count verification gates old-prefix
  deletion. One extra failure mode from reusing the inactive annotation: a migrator
  that dies between fencing and flipping leaves the workspace `Inactive`; the resumed
  migrator (or a janitor) must detect the stale fence via the in-progress
  `identityHashes` state and either complete the flip or lift the fence.
* **Permission-claim breakage on rotation.** Claims reference the identityHash, so
  rotation invalidates them by definition. The alias with canonical-hash normalization
  (see the cross-export claims section) keeps stale references working through the
  deprecation window and surfaces them via `ClaimIdentityRotated` conditions, rather
  than letting claims silently stop matching; breakage only occurs after the alias is
  deliberately retired (or immediately, if the revoke flag was chosen).
* **Long-running rotations on exports with many consumers.** Rotation is per-workspace
  incremental and resumable; `migratedBindings/totalBindings` makes progress visible,
  and a rotation can be paused (stop fanning out) without leaving any workspace in a
  broken state — each workspace is either fully old-identity or fully new-identity.
* **Identity sprawl going unnoticed.** The `IdentityShared` condition makes
  multi-export identities visible on every affected export, turning archaeology into a
  queryable signal for when to rotate.
* **The alias window keeps the old hash honored.** By design — it is what prevents
  cross-export claims from going dark — but it means a rotation is not fully effective
  until the alias is retired. For security-motivated rotations the immediate-revoke
  flag skips the window and accepts the claim breakage. Either way, the export status
  shows whether an alias is still live, so "rotation done" is never ambiguous.
