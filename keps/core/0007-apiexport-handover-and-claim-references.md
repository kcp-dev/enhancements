# APIExport Storage Handover and Claim References

Builds on: [0006 APIExport Identity Decoupling](0006-apiexport-identity-decoupling.md).
Everything here assumes 0006's model: the identity is an immutable public identifier, the
storage prefix is tracked per bound resource as `storage.key`, and re-keying moves data
without touching the identity.

Related: [0004 APIBinding Resource Adoption](0004-apibinding-resource-adoption.md), whose
zero-copy split needs §1 once identity secrets are gone;
[Identity-Agnostic Permission Claims](https://github.com/kcp-dev/enhancements/pull/22),
which gates claims that name no identity at all and overlaps with §3.

Status: draft, split out of 0006 to keep that proposal about decoupling only. A draft of
all three parts exists on the kcp branch `https://github.com/kcp-dev/kcp/pull/4484` and
will be separated from the 0006 implementation.

## Summary

Three additions on top of 0006, independent of each other:

1. **Explicit storage handover.** An export may serve a resource out of another export's
   prefix, with that export's consent: `storage.adopted` on the adopter, `crd.sharedWith`
   on the owner. This is the zero-copy 0004 split for a world without identity secrets.
2. **Claims by export reference.** A claim may name the claimed export, `export: {path,
   name}`, instead of pasting its hash.
3. **Claim grants.** The claimed export may list who is allowed to claim a resource up
   front (`claimableBy`), satisfying its maximal permission policy and optionally skipping
   per-consumer acceptance.

## Motivation

- **Splitting an export copies data.** Under 0006, once `secretRef` is gone the only way to
  move a resource from export A to export B is a re-keying: every consumer workspace is
  fenced and copied. 0004 made the same operation zero-copy by sharing the identity, and
  0006 removes that option. A handover restores it without a shared identity.
- **Pasting hashes is error-prone.** A claimer has to find the claimed export's
  `status.identityHash` and copy it. Naming the export is what the claimer actually means.
- **Consent is one-sided.** Today a claim is accepted by each consumer and checked against
  the claimed export's maximal permission policy, but the claimed export has no way to say
  "this claimer is fine" once, for every consumer.

### Goals

- A resource can be handed from one export to another without copying objects, with the
  owner's explicit consent, and the handover is visible on both exports and in the registry.
- A claim can name an export; kcp resolves the identity and records it.
- A claimed export can grant claims by export, by label selector or by RBAC.
- None of this changes identities, the etcd layout or the `resource:identity` addressing.

### Non-goals

- Changing how identities are assigned or converted (0006).
- Gating claims that name neither a hash nor an export (PR #22).
- Filtering wildcard reads over a shared prefix (see Prerequisites).

## Prerequisites

- **0006 merged.** `storage.key` per bound resource, one bound CRD per storage key on a
  shard (0006 §4) and re-keying are what a handover rests on.
- **Wildcard filtering over shared storage.** While B's bindings sit on A's prefix, each
  provider's virtual workspace sees the other's customers' objects for that resource. 0006
  accepts this for converted legacy pairs because they are grandfathered and time-boxed.
  A handover creates that state on purpose, so filtering wildcard list and watch by
  "workspace is bound to this export" must land first or together with §1.
- **A decision on PR #22.** §3's `exportSelector` and `rbac` grants and PR #22's
  `PermissionClaimPolicy` both answer "who may claim this without a hash". One of them
  should be the gate; the other should be expressed in its terms or dropped.

## Design

### 1. Explicit storage handover

An export declares storage per resource. Today `spec.resources[].storage` is one of `crd`
or `virtual`. This adds a third kind, `adopted`: a bound CRD that lives on another export's
prefix. Consent sits on the owner's side of the same field.

```yaml
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport
metadata:
  name: networking-v2
spec:
  resources:
  - name: vnets
    group: networking.io
    # No schema: an adopted resource is served with the source's schema,
    # because the objects on the shared prefix were written by it.
    storage:
      adopted:
        from: {path: root:org:networking, name: networking}
  - name: routes
    group: networking.io
    schema: v260301-5b9d7a0.routes.networking.io
    storage:
      crd:
        # Exports that may serve `routes` out of this export's prefix.
        # This is a handover, not access; access is a claim.
        sharedWith:
        - {path: root:org:networking, name: networking-edge}
status:
  identityHash: 2cynbfy2m0wtjqcs-8f1c0a7e...
  resources:                       # schema in effect per resource
  - {group: networking.io, resource: vnets, schema: v250210-8c1e4f2.vnets.networking.io, schemaCluster: 1k3f...}
  storage:                         # prefixes bindings of each resource use
    resources:
    - {group: networking.io, resource: vnets, keys: [1k3f...-3b9d44c1...]}   # networking's
    - {group: networking.io, resource: routes, keys: [2cynbfy2m0wtjqcs-8f1c0a7e...]}
  conditions:
  - type: ResourcesValid           # every adopted resource found its source and consent
  - type: StorageShared            # some prefix is shared, in either direction
```

In plain words, with `VirtualMachine` as the resource: provider A exports VMs, customers
bind A and create VMs, and all those VMs sit in one storage room labelled with A's
identity. A later hands the VM product to provider B. Instead of moving every VM into B's
room, A gives B a key to the VM room (`sharedWith`) and B accepts it (`adopted`). Both work
on the same VMs in place; nothing is copied. B's other resources stay in B's own rooms. B
keeps its own name on its own door, and that name is what claims and URLs refer to. Later,
when A wants its room back, B's customers' VMs are carried into B's room by a re-keying.
Taking B's key away without carrying anything over would lock B's customers out of their
own VMs, so kcp does not offer that; removing an export from `sharedWith` leaves existing
bindings on their prefix and flags the adopter (`ResourcesValid: StorageNotShared`) until
the owner re-keys.

**Handover or claim?** Compute needs to read and create vnets to place VMs: that is a
claim. Networking replaces its export with `networking-v2`, which should serve the existing
vnets in place: that is a handover.

| | Claim | Handover (`adopted` / `sharedWith`) |
|---|---|---|
| What the other export gets | access to A's resource in A's consumers' workspaces | the right to serve one resource it also exports, out of A's prefix |
| Who exports the resource | only A | both; a workspace binds one or the other |
| Who consents | each consumer, plus A's maximal permission policy or a grant (§3) | A's owner, once |
| What is visible | objects in accepting workspaces, filtered by labels | everything under the prefix (see Prerequisites) |
| Lifetime | permanent | until re-keyed |

Rules:

- An adopted resource must be served by the source with `crd` storage and the source must
  list the adopter in `sharedWith`; chains of adoption are rejected. Until consent appears
  the adopter's bindings wait with reason `StorageNotShared`.
- `crd` may become `adopted` only before the export has an identity; afterwards data lives
  under the identity and only a re-keying may move it. `adopted` may become `crd` at any
  time; existing bindings keep their prefix until re-keyed.
- Self-adoption and more than one storage kind on a resource are rejected by admission.
- The registry's `APIIdentity.status.storageSharedWith` lists adopters, and `StorageShared`
  is set on both exports.
- 0004's successor verification for binding deletion is by storage prefix and the schema in
  effect, so an adopting export is a valid successor although its identity differs.

### 2. Claims by export reference

A claim may name the claimed export instead of its hash.
We might even make explicit export primary and deprecate the use of identity hashes in claims.

```yaml
# the claiming export on APIBinding. 
# This is the case when Binder wants to claim a resource from another export but exports don't have
# cross-claim references.
spec:
  permissionClaims:
  - group: networking.io
    resource: vnets
    export: {path: root:org:networking, name: networking}   # instead of identityHash
    verbs: ["get", "list", "watch", "create"]
```

kcp resolves the reference to the export's identity and records it in a new
`status.permissionClaims[]` list with `identityHash` and `resolvedBy: Export`. A reference that
matches nothing makes the claim `ClaimUnresolved` and inert, reported on the claiming
export's `ClaimsResolved` condition. Built-in types and ClusterCachedResources have no
export to name and keep using `identityHash`.

A reference is a convenience, not consent: the claim still needs each consumer's
acceptance and passes the claimed export's maximal permission policy.

### 3. Claim grants

The claimed export may list who is allowed to claim a resource:

```yaml
# the claimed export on APIExport
spec:
  resources:
  - name: vnets
    group: networking.io
    schema: v250210-8c1e4f2.vnets.networking.io
    storage: {crd: {}}
    claimableBy:
    - export: {path: root:org:compute, name: compute}
      verbs: ["get", "list", "watch", "create"]       # cap on the claim's verbs
    - exportSelector: {matchLabels: {tier: platform}}  # every export with these labels
    - rbac: {}                                         # every export whose owner is allowed
                                                       # verb "claim" on this APIExport
      delegated: true                                  # see below
---
# the claiming export: no hash, no export reference
spec:
  permissionClaims:
  - group: networking.io
    resource: vnets
    verbs: ["get", "list", "watch", "create"]
```

A claim with neither `identityHash` nor `export` resolves through grants, recorded with
`resolvedBy: Grant`. More than one granting export makes the claim `ClaimAmbiguous` until
it adds `export` or `identityHash`.

A grant is the claimed export saying "I trust this export with my resource". It satisfies
the claimed export's maximal permission policy and caps the verbs. It does not decide for
the consumer: each consumer still accepts the claim on their APIBinding, by group and
resource, with the resolved identity shown in the binding's status. `delegated: true` lets
the claimed export extend its own access to the grantee, so consumers bound to both exports
need not accept.

Grants are never required. Two hundred producers claiming `workspaces.tenancy.kcp.io` add
nothing to the root export; each names the identity and consumers accept.

RBAC grants evaluate a SubjectAccessReview in the granting workspace as user
`apis.kcp.io:export:<cluster>:<name>`, group `apis.kcp.io:exports`, verb `claim`. Grants
are indexed, so resolution is a lookup.

## API changes

- `ResourceSchemaStorage` gains `adopted: {from: {path, name}}`; exactly one of `crd`,
  `virtual`, `adopted`. `ResourceSchemaStorageCRD` gains `sharedWith: [{path, name}]`.
- `ResourceSchema.schema` becomes optional for adopted resources. `ResourceSchema` gains
  `claimableBy: [{export | exportSelector | rbac, verbs, delegated}]`.
- `PermissionClaim.identityHash` becomes optional; `PermissionClaim` gains `export: {path,
  name}`.
- `APIExportStatus` gains `permissionClaims: [{group, resource, identityHash, resolvedBy,
  delegated, verbs}]`, `resources: [{group, resource, schema, schemaCluster}]` and the
  conditions `ResourcesValid`, `ClaimsResolved`.

## Security

A handover is full data sharing for one resource while it lasts: granted only by the
owner, only to named exports, visible as `StorageShared` on both sides and in the registry.
Admission enforces consent on `adopted`, so no export can place itself on another's prefix.

Knowing an identity or an export's path is not consent. A claim by reference or by grant
still needs the consumers' acceptance; a grant only replaces the maximal permission policy
check with the owner's explicit statement. `sharedWith` (storage) and `claimableBy`
(access) stay separate fields on purpose.

## Open questions

- Should the handover be one-sided (`sharedWith` alone) to simplify 0004 tooling?
  Two-sided is proposed because it mirrors claims and leaves an audit trail on the
  adopting side.
- Should `delegated` be the default for grants? A grant already means the claimed export
  trusts the claimer, and the claimed export has that access anyway. The draft keeps
  per-consumer acceptance as the default for visibility.
- Should §3 be expressed through PR #22's `PermissionClaimPolicy` instead of per-resource
  grants, or the other way round?
- Adopter-owned schema on a shared prefix needs a storage-compatibility check; until then
  the adopter gets its own schema by re-keying onto its own prefix. Is that acceptable for
  the first iteration?
