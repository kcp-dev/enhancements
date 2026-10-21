# Identity-Agnostic Permission Claims

## Summary

An `APIExport` that wants to claim another provider's API must today write that
provider's **identity hash** into the claim:

```yaml
permissionClaims:
- group: ai.example.ai
  resource: models
  identityHash: 5fdf7c7aaf407fd1594566869803f565bb84d22156cef5c445d2ee13ac2cfca6
  verbs: ["get", "list", "watch"]
```

This works for a single, known producer. It does not work for a **platform**: a
family of APIs where `infrastructure.example.ai` should be able to claim
`ai.example.ai` and `edges.example.ai` for every consumer that uses them,
without anybody enumerating identities up front, and where the same
group/resource may legitimately be served by different producers in different
consumer workspaces — a central platform's producer in some, a user's
self-hosted producer in others — or by two identities at once while an
identity rotation drains.

This proposal makes `identityHash` optional for claims that a new
installation-wide policy object allows. Such a claim is **identity-agnostic**:
it names only `group` and `resource`, and resolves, *per consumer workspace*, to
whatever identity that workspace's `APIBinding` for the claimed resource
carries. Nothing — not the claimer, not the policy, not the consumer — ever
names an identity hash.

The gate is a new cluster-scoped `PermissionClaimPolicy`, stored in the **Admin
workspace** (`/services/admin`, see
[The Admin Workspace](../sharding/shard-aggregated-admin-view.md)) and therefore
installation-wide and reachable identically through every shard. It answers two
questions with explicit lists and no wildcards:

- which API groups an APIExport exporting a given "claimer" group may claim
  without an identity hash, and
- who may export the API groups the policy names at all.

Related: [0005 — APIExport Identity Rotation](0005-apiexport-identity-rotation.md).
Rotation is what makes a *set* of identities, rather than one, the honest answer
for a claimed resource; this proposal resolves that set per workspace at request
time, so a claim keeps working across a rotation with no edit.

## Motivation

### The identity hash is a property of a binding, not of a claim

kcp uses the identity hash for two different jobs:

1. **Disambiguation at bind time.** An `APIBinding` names the identity hash so
   the consumer provably binds the intended producer of `widgets.example.com`
   and not an impostor's export of the same group.
2. **Data segregation in storage.** Bound instances live under
   `/registry/<group>/<resource>/<identityHash>/<cluster>/...`, and a wildcard
   request carries the hash in the resource name
   (`/clusters/*/apis/<group>/<version>/<resource>:<identity>`), so an export
   owner sees exactly their own consumers' objects.

Job 1 is settled *by the consumer, in the consumer's workspace*, when they bind.
By the time a permission claim is evaluated, the consumer workspace already
knows which producer serves `models.ai.example.ai` there. Forcing the *claimer*
to restate that answer, for every consumer, in advance, is the mismatch this
proposal removes.

Job 2 still needs a hash, and this proposal does not change storage at all. It
changes *where the hash comes from*: from the claim's spec to the consumer's
binding.

### Use cases

**1. A platform family.** One operator runs a family of related APIs —
`infrastructure.example.ai`, `ai.example.ai`, `edges.example.ai` — and the
infrastructure controller needs to read the `ai` and `edges` objects in every
consumer workspace that uses them. The claim is a statement about the family
("infrastructure may claim ai"), not about a particular export or hash.

**2. Self-hosted providers next to a central platform.** The platform offers
`ai.example.ai` centrally, but some users deploy and run their own, self-managed
producer of the same API in their own workspace — for data residency, their own
hardware, or a fork with local changes. Those users still consume other APIs
from the central platform, such as `infrastructure.example.ai`, and expect its
claim on `ai.example.ai` to work against *their* producer exactly as it does
against the central one for everybody else. The central claimer cannot know
these self-hosted identities in advance: they appear whenever a user sets one
up, and each user's producer has its own hash.

Today neither case can be expressed. Case 2 is the sharper one: even a claimer
willing to maintain a list of hashes would have to learn about, and be edited
for, every self-hosted deployment.

Case 2 also shows why reservation (below) matters. A self-hosted producer of
`ai.example.ai` is legitimate only because the installation admin says so, by
admitting its owner — directly, or through a group such as
`example-self-hosted-providers` — into the policy's `providers`. Without that,
anyone could stand up an "`ai.example.ai`" and have the central claimer read it.

### What goes wrong today

**One claim, one producer.** A claim holds exactly one hash, so it only sees
one producer. If a second real producer of `ai.example.ai` exists, the claim
cannot see it.

**Rotation breaks claims.** While a producer rotates its identity (KEP 0005),
some consumers use the old hash and some use the new one. A claim with either
hash misses half of them until the rotation ends, and then someone has to edit
it by hand.

**Failures are silent.** If the claim's hash does not match what the consumer
bound, the consumer still sees their objects marked as claimed, but the
provider's controller gets an empty list. Nobody gets an error. See kcp#2152.

### Why an allowlist of identities is not the answer

The obvious smaller fix is to let `identityHash` be a list. It does not help:
the claimer still has to know every hash, still has to edit the claim on every
rotation and every new producer, and the virtual workspace still needs the
multi-identity serving path that is the expensive part of the work. Having paid
for that path, deriving the set is strictly better than authoring it.

### Goals

- A permission claim may name only `group` and `resource`, and resolve to the
  right identity in each consumer workspace.
- Nobody — claimer, consumer, platform admin — needs to know an identity hash.
- Multiple genuine producers of the same group/resource, in different consumer
  workspaces, are all reachable through one claim.
- An identity rotation of a producer requires no change to any claim.
- Which groups may be claimed this way, and by whom, is explicit, listed, and
  installation-wide. No wildcards.
- The producer's `maximalPermissionPolicy` stays in force for these claims.

### Non-Goals

- Changing the etcd key layout, the wildcard request format, or the meaning of
  the identity hash in storage.
- Removing identity hashes from claims. A claim that names a hash keeps its
  exact current behaviour and remains the right choice for a single known
  producer.
- Cross-shard aggregation. As today, a shard's virtual workspace serves what
  that shard holds; `APIExportEndpointSlice` remains how a provider reaches
  every shard.
- Any automatic trust between groups. A policy is written by an installation
  admin; nothing derives one from a shared DNS suffix.

## Proposal

### 1. `PermissionClaimPolicy`

A new kcp-scoped type in `admin.kcp.io/v1alpha1`:

```yaml
apiVersion: admin.kcp.io/v1alpha1
kind: PermissionClaimPolicy
metadata:
  name: example
spec:
  providers:
  - kind: Group
    name: example-providers
  claims:
  - claimer: infrastructure.example.ai
    groups:
    - ai.example.ai
    - edges.example.ai
  - claimer: ai.example.ai
    groups:
    - edges.example.ai
```

**`claims[].claimer` identifies the claimer by the API group it exports**, not
by name, path or workspace. An APIExport is "the infrastructure export" if it
serves a resource in group `infrastructure.example.ai`. Claimers and producers
may live anywhere in the installation; no path is ever mentioned.

**`spec.providers` are the subjects allowed to export any group the policy
names** — claimer groups and claimed groups alike. This is the reservation that
replaces what the identity hash used to guarantee (see
[Why reservation is the security anchor](#why-reservation-is-the-security-anchor)).
Subjects are `User` or `Group`; service accounts match through their user name
or their `system:serviceaccounts*` groups.

Both lists are exact strings. There is deliberately no pattern syntax: an admin
adding `metrics.example.ai` to the family edits the policy, and that edit is
the review point.

Policies compose asymmetrically, and deliberately so. A claim is allowed if
**any** policy allows it, so adding a policy can only grant more claiming. A
reserved group is exportable only if **every** policy reserving it admits the
subject, so adding a policy can only narrow who may export. In both directions a
new policy is the conservative move: it never silently widens the ability to
serve a reserved group. The "all must admit" rule mirrors how overlapping
maximal permission policies already compose.

### 2. Storage: the Admin workspace

Policies are installation-wide, so they must not live in `root` (which the
[de-rooting work](../sharding/shard-aggregated-admin-view.md) is removing as a
special place) and must not be per-shard. They are stored in the **cache
server**, under a synthetic logical cluster reserved for objects the Admin
workspace owns, and served for read and write through `/services/admin`:

```
kubectl ws use :admin
kubectl get permissionclaimpolicies
kubectl apply -f example-policy.yaml
```

This is the Admin workspace's first *owned* resource. `Shard` objects there are
an aggregated view of objects the shards own; a `PermissionClaimPolicy` has no
authoritative copy anywhere else. Consequences:

- Every shard reads policies through its existing cache-backed informers. No
  replication controller, no per-shard copy, no root dependency.
- Writing requires `system:kcp:admin` membership, which the Admin workspace
  already enforces. It is extended to admit creation and deletion for this
  resource only; `Shard` stays read-plus-allow-listed-fields.
- The cache server serves the new type alongside the others it already holds.

Note that access to the Admin workspace is **group membership only** — it admits
`system:kcp:admin` (and the privileged `system:masters`) and denies everything
else, with no delegation to RBAC. So policy management cannot today be granted
to a narrower identity, for example a platform-operator service account that
manages policies but nothing else. That limitation is not introduced here and is
already visible for shard administration; it is the same gap that makes a
least-privilege front-proxy identity impossible against `/services/admin`.
Whatever fix lands there — delegating to RBAC, or defining purpose-specific
groups — applies unchanged to this resource. Until then,
`PermissionClaimPolicy` is an installation-admin object, which for a first
iteration is the right blast radius anyway.

### 3. Admission on `APIExport`

Admission on `APIExport` becomes policy-aware. Two rules:

**Identity-less claims.** For each `spec.permissionClaims` entry with an empty
`identityHash` that is not a built-in API and not in `apis.kcp.io`, the export
must itself export some group `C` such that a policy has a `claims` entry with
`claimer: C` whose `groups` contain the claimed group. Otherwise the write is
rejected with today's error, extended to say which groups the export serves and
what it tried to claim.

**Reserved groups.** For each `spec.resources` entry whose group is named
anywhere in any policy, the requesting user must match that policy's
`providers`. The check runs only on groups the write *adds* — the set of
reserved groups on the old object is subtracted first — so an unrelated status
or schema update by a less privileged controller keeps working.

Until policies are visible to a shard, none is in effect: identity-less claims
are rejected exactly as before this feature, and reservation is not enforced.
Fail-closed on the new capability, no behaviour change for existing objects.

Pre-existing exports in a group that a policy later reserves are not
retroactively rejected; admission only sees writes. This is called out in
[Risks](#risks-and-mitigations).

### 4. Runtime: resolution per consumer workspace

For a claiming export `C`, a claimed group/resource `GR`, and a consumer
workspace `W`, the identity is **whatever `W`'s `APIBinding` serving `GR`
carries in `status.boundResources[].schema.identityHash`**. Two questions are
answered from the `APIBinding` objects a shard already holds:

- **In one workspace:** the identity `W` bound for `GR`.
- **Across the shard:** the deduplicated set of identities over every workspace
  that holds a binding to `C` with an accepted identity-agnostic claim for `GR`.
  A binding may reference its export by canonical path or by cluster name, and
  both forms resolve to the same export.

A consumer that accepted the claim but has no binding serving `GR` contributes
nothing, which is correct: nothing is claimed there, and nothing is labelled
there either.

Nothing is cached. The set is read at request time, so a rotation, a new
consumer or a new producer is picked up without rebuilding anything.

### 5. Serving: multi-identity requests

The APIExport virtual workspace stops carrying one fixed identity for a claimed
resource and instead asks, per request, which identities that resource is stored
under on this shard. Given that set:

| request | no identity | one identity | several identities |
|---|---|---|---|
| per-cluster (get, create, update, patch, delete) | plain resource name | `resource:hash` | plain resource name |
| wildcard list | empty list, shard not called | as today | one list per identity, concatenated |
| wildcard watch | empty watch, stays open | as today | one watch per identity, merged |

The per-cluster "several identities → plain name" row is the important one. With
no `:identity` suffix the shard resolves the identity from the binding in the
target cluster, which is precisely the answer this proposal wants. The
multi-identity path is therefore only needed for wildcard traffic.

For a wildcard list, the returned `resourceVersion` is the numeric maximum
across the per-identity responses — all identity prefixes live in the same etcd
on the shard, so revisions are comparable — and pagination is disabled while more
than one identity is in play. An identity that no binding serves on this shard
is skipped, exactly as a single unserved identity already is.

The merged watch stops every underlying watch when the caller stops or the
request ends, and ends the whole watch if any one of them ends, so a consumer
relists rather than silently losing one identity's events.

### 6. Authorization

`maximalPermissionPolicy` must keep working, and this is the one place where
getting it wrong would be a real hole. Today an empty `identityHash` on a claim
is read as "unclaimable resource, no policy can exist" and **allowed**. That is
correct only for built-ins and `apis.kcp.io`.

The two cases are now distinguished. For an identity-agnostic claim the identity
is resolved — the one bound in the request's workspace, or, for a wildcard
request where no single workspace can be named, the whole set — and every
resulting producer's policy must allow, exactly as it already must when several
exports share one hash. If nothing serves the claimed resource there yet, the
request is **denied** and the caller retries, rather than falling through
unchecked.

Checking that the consumer accepted the claim at all needs no change: it matches
the consumer's accepted claim against the export's claim, and an empty hash on
both sides matches.

### 7. What needs no change

Worth stating explicitly, because it is most of the system:

- **Claim labels.** The label identifying a claimed object is derived from the
  claim including its (empty) identity hash, and both the export side and the
  binding side carry the same empty value, so the two agree. This is the
  divergence class of kcp#4198 and it does not arise here.
- **Bound CRDs for claimed resources.** An identity-agnostic claim resolves
  *through* an `APIBinding` in the consumer workspace, and that binding already
  brings the claimed resource's CRD onto the shard.
- **Storage, rotation, CRD cleanup, cache replication.** Untouched.

## API Changes

New type `apis.kcp.io/v1alpha1 PermissionClaimPolicy` (cluster-scoped, no
status in v1alpha1). `spec.claims[]` is a list-map keyed by `claimer`;
`spec.claims[].groups` is a set; `spec.providers[]` is atomic.

`APIExport.spec.permissionClaims[].identityHash` is unchanged in schema — it is
already optional with an empty default. Only admission's interpretation of an
empty value changes, and only in the direction of allowing more.

No other API changes shape.

## Rollout

The feature is inert until an admin creates a policy: with no policy, admission
rejects identity-less claims exactly as it does today and no resolution path is
reachable. There is no migration and no flag. An installation that never
creates a `PermissionClaimPolicy` behaves identically to one without this
change.

## Why reservation is the security anchor

> **Assumption: the platform administrator is in full control of identities.**
> Identity-agnostic claims are only as safe as the policy that gates them. This
> proposal assumes the installation admin decides, and keeps deciding, exactly
> who may export each reserved group — including every self-hosted provider
> admitted into `providers`. If that control is weak (a `providers` group whose
> membership is loosely managed, a shared service account, a stale entry for a
> user who should no longer produce the API), then whoever slips in becomes a
> genuine producer as far as every claimer is concerned, and claimers will read
> and act on their objects. In installations where the admin cannot guarantee
> this, claims should keep naming an explicit `identityHash`.

Without `providers`, identity-agnostic claims would be unsafe, and the reason is
worth spelling out.

Anyone may create an `APIExport` for group `ai.example.ai` in their own
workspace. Suppose consumer `W` binds an impostor's `ai.example.ai` export and
accepts the infrastructure claim. Resolution would hand the claiming controller
the impostor's objects under the real schema's name. The identity hash used to
prevent this: the claim named one specific producer, and only that producer's
objects were ever visible.

`spec.providers` restores the guarantee at the other end. Because only
authorized subjects can create an APIExport in a reserved group, *every* binding
of `ai.example.ai` anywhere in the installation is a binding of a genuine
producer. Per-workspace resolution then needs no further check: there is no
impostor to resolve to.

This generalises an existing mechanism. kcp already reserves `*.kcp.io` for
itself and refuses to let anyone else define APIs in it.
`PermissionClaimPolicy` makes reservation data instead of a hard-coded rule.

Two consequences follow, and both are intentional:

- Whoever can export the claimer group can claim everything listed under it.
  That is the definition of a platform family. It argues for one policy per
  provider family rather than one global policy, so grants stay narrow.
- Reserving a group an installation already uses freezes further exports of it
  to the listed providers. See [Risks](#risks-and-mitigations).

## Alternatives Considered

**Multi-valued `identityHash` on the claim.** Rejected: the claimer must still
know every hash and edit on every rotation, and it needs the same multi-identity
serving path, so it costs nearly as much and solves less.

**Mutating admission that stamps a resolved hash into the claim's spec.** Much
cheaper — no runtime change at all — but it pins one identity at write time,
which is exactly what rotation and multiple producers break. It also requires
the producer export to exist before the claimer is written.

**A group-to-canonical-export allowlist** (policy names the one export each
group resolves to). Simpler to implement, but it reintroduces a single canonical
identity per group and so fails both the rotation case and the
multiple-producer case that motivate this KEP.

**An `exportRef` on the claim**, resolving identity by path like `APIBinding`
does. This is a good *complementary* feature for the single-known-producer case
and could be added later; it does not serve the platform case, where the
producer differs per consumer.

**Identity-agnostic wildcard on the shard.** Instead of fanning out in the
virtual workspace, teach the shard to serve a full-object wildcard across every
identity of a group/resource, the way it already serves wildcard *partial
metadata* today — that identity-less view is how the claim labeler and garbage
collector see all objects of a group/resource. That is the cleaner end state and
would remove the fan-out entirely. It is deferred because such a request spans
schemas from different producers with different storage versions, which is new
ground. The claimer-facing API is identical either way, so it can replace the
fan-out later without touching a single `APIExport` or `APIBinding`.

**RBAC verbs (`claim` and `provide`) on the policy object instead of
`claims[].claimer` and `spec.providers`.** Considered and partly adopted in
spirit: `providers` is an in-object subject list rather than an RBAC verb
because it must be evaluated inside APIExport admission on every shard, where a
delegated SubjectAccessReview against the Admin workspace would add a
cross-component dependency to a hot write path. An RBAC-verb variant remains
open if per-policy delegation becomes necessary.

## Risks and Mitigations

**A claim resolves to nothing and fails open.** Mitigated by denying when no
identity resolves, and by serving nothing — rather than everything — in the
virtual workspace.

**Reserving a group that is already in use.** Creating a policy for a group that
existing exports already serve does not invalidate them — admission sees only
writes — but it does block *new* exports of that group by non-providers, and it
blocks those existing exports from adding resources in reserved groups. An admin
creating a policy for an in-use group should audit existing exports of it first.
A follow-up could surface a status condition listing exports in reserved groups
that predate the policy.

**Discovery of a claimed resource becomes conditional.** A claim that names an
identity hash finds the claimed resource's schema through that identity, so the
resource appears in the virtual workspace's discovery on every shard, serving an
empty list where no consumer exists. An identity-agnostic claim finds its schema
through a resolved identity, so on a shard where no consumer has yet bound the
claimed resource it is not served at all, and appears once the first consumer
binds. A provider controller that builds informers from discovery will therefore
see the resource arrive rather than start empty.

The alternative is to look the schema up by exported group/resource instead and,
when no identity resolves, take it from any export serving that group while
still serving no data (no identities already means "empty list, shard not
called"). Because reservation guarantees every such export is genuine, picking
one for the schema is safe — it is the same "pick one, same owner" rule that
already applies when several exports share an identity. This is deliberately
left out of the first implementation to keep the change small, and is the first
thing to add if conditional discovery proves awkward in practice.

**Wildcard fan-out cost.** N identities means N watches per claimed resource per
claiming export on a shard. In practice N is 1 in the steady state and 2 during
a rotation; it is the number of *distinct producers* on the shard, not the
number of consumers. If a deployment ever sees large N, the shard-side
identity-agnostic wildcard above removes the fan-out entirely.

**Pagination on multi-identity wildcard lists.** Disabled while more than one
identity is in play. A very large claimed collection would return in one
response. Acceptable initially because that window is the rotation window; a
composite continue token is the fix if needed.

**Admin workspace becomes a write path.** Until now `/services/admin` accepted
only allow-listed field updates on objects the shards own. It now creates and
deletes objects of its own. Mitigated by scoping creation and deletion to
admin-owned resources by name, and by the existing `system:kcp:admin`
requirement.

**A stale cache pins an old identity.** Resolution holds only the claiming
export's coordinates, re-reads the export on each request, and re-derives the
set from the bindings then, so no resolved value is ever held across requests.

## Open Questions

- Should `PermissionClaimPolicy` carry a status listing the exports currently
  exporting each reserved group, so an admin can see who a new policy would
  affect before applying it?
- Should a claimer with no exported resources be supported (an export that only
  claims)? Today it has no group to be identified by. A `claim` RBAC verb on the
  policy is the natural fallback for that case.
- Self-hosted providers (use case 2) must each be admitted into `providers`,
  which makes the installation admin a gate on every new self-hosted
  deployment. Is a group the admin manages enough, or should a policy be able
  to delegate provider enrollment, e.g. to the platform operator?
- Should `claims[].claimer` optionally accept a resource, not just a group, for
  finer identification?
- Ordering with KEP 0005: during a rotation the resolved set legitimately holds
  two hashes. Should the virtual workspace surface that (a condition, a metric)
  so operators can see a drain in progress through the claim's eyes?
