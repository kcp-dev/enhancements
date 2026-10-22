# Decouple API Identity from Provider Credentials Enhancement Proposal

## Summary

The APIExport `identityHash` is derived from the secret used to
authorize serving it.

This `identityHash` is also used as the API identity and as part of the
storage key in the consumers' etcd storage.

This enhancements aims at decoupling the API identity and the provider
credentials used to serve it to allow rotating credentials without
requiring intensive operations on the consumer side.

This is improves upon [APIExport Identity Rotation](0005-apiexport-identity-rotation.md)
by decoupling the identity and secrets.

### Goals

- Decouple the API identity from the secrets used to authorize serving it.
- Serve the same API from multiple workspaces without using the same credentials.
- Allow user-chosen storage keys.
- API identity must be assumed and documented to be publicly known.

### Non-Goals

- Implementing new API serving authorization mechanisms, e.g. OIDC-based
  instead of a secret in the workspace or by the cluster name the
  APIExport originates from.
- Moving consumer data.

## Proposal

### Terminology

1. **API identity** is what used to be the APIExport `identity hash`.
   In discussions this was also referred to as the "storage key" to
   focus on the storage layer, however the term `identity` is already
   used by consumers and it will be less confusing to continue using it
   to specify the APIs identity.

   This is publicly known and used as the storage key in the consumer
   and as part of resource URLs.

   The identity is immutable.

2. **Provider credential** the secret in the `kcp-system` today.

    Credentials must not be immutable and must be swappable without
    major impact.

3. **credential hash** is the hex-encoded lowecase sha256 hash of a provider credential.
   This is the `identity hash` today.

### APIExport API

The APIExports `status.identityHash` is deprecated and no longer read
except for filling `spec.identity.value` on upgrade.

```yaml
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport
metadata:
  name: widgets
spec:
  identity:
    # current ref
    secretRef:
      namespace: kcp-system
      name: widgets

    # New APIExport identity.
    # If unset filled with the current `status.identityHash`.
    # If unset and `status.identityHash` is empty a random 64 character alphanumeric string is generated.
    value: <alphanumeric string>

  secretRef:
    # must be in kcp-system
    name: widgets
    # if unset falls back to `spec.identity.secretRef`

  # additional credentials allowed to serve the same API identity
  writers:
    - credentialHash: <hex-sha256-of-next-secret-data-key>

  resources: []
  permissionClaims: []
status:
  # marked deprecated, unused
  identityHash: <hex-sha256-of-current-secret-data-key>

  # written by controller after validating, either `spec.identity.value` or empty
  # protected by admission
  identity: <value of .spec.identity.value>

  # status.writers is computed from .spec.secretRef and concat'ed with .spec.writers
  # written by controller after validating
  # protected by admission
  writers:
    - credentialHash: <hex-sha256-of-current-secret-data-key>
    - credentialHash: <hex-sha256-of-next-secret-data-key>
```

The `spec.identity.value` is immutable once set.

### Authoritative Registration

The API identity is used in a couple places to identify an API.

E.g. the virtual workspace code assumes that the same identity and GVR
means same owner:

- https://github.com/kcp-dev/kcp/blob/b16875623ece22adfed221e1d5ddbd307cc4aa56/pkg/virtual/apiexport/controllers/apireconciler/apiexport_apireconciler_reconcile.go#L156-L164
- https://github.com/kcp-dev/kcp/blob/b16875623ece22adfed221e1d5ddbd307cc4aa56/pkg/virtual/apiexport/authorizer/maximal_permission_policy.go#L102-L105

That means an API identity must be globally unique in a kcp instance.

The cache-server is used via the `APIIdentity` below to declare an
APIExport as the owner of an identity.

In the future a new kcp component is needed, e.g. `kcp-registry` that is
a global authoritative database for data like this.

The shards validate new APIExports against the APIIdentity as mentioned below.

### `registry.kcp.io`

To synchronize API identity claiming the shard creates
`APIIdentity.registry.kcp.io` in a shared scope in the cache-server with
the name being the identity.

```yaml
apiVersion: registry.kcp.io/v1alpha1
kind: APIIdentity
metadata:
  name: <spec.identity.value>
spec:
  owner:
    ref:
      cluster: a1b2c3
      name: widgets
      uid: 1234
status:
  # filled from the status.writers of the owning APIExport
  writers: []
```

This allows an APIExport to claim this API identity globally across the
kcp instance.

The writers in the status are updated from the owning APIExport.

When the APIExport is deleted the APIIdentity it owns is transferred to
the next oldest authorized APIExport based on the `status.writers`.

When no other authorized APIExports exist all shards with consumers of
the APIExport, determined by the APIExportES, are requeuing the
APIIdentity until all consumers and their objects are deleted.

Only once all consumers are gone the APIIdentity is freed for reuse.

As a consequence a provider also cannot reclaim an APIIdentity with
active consumers if they deleted the APIExport.

When an APIExport is created the controller first tries to claim the APIIdentity for it.

If the APIIdentity does not exist it is created and the APIExport is entered as the owner.

If it does exist the `status.writers` is retrieved and validated against the APIExports `spec.secretRef`.

If the `spec.secretRef` matches any of the `status.writers` (e.g. the
referenced secrets hex-sha256 sum matches an entries `credentialHash`)
the APIExport is allowed to serve an API with that identity.

If that is not the case the APIExport goes into a failstate is not
requeued.

A separate reconciler watches the APIIdentities in the cache-server in
the shared scope. When the `status.writers` changes it rechecks and
validates the APIExports on the shard.

As a consequence every reconciler using an APIExport must check
`status.identity` and refuse handling it when `status.identity` is
empty.

#### Follow-up: Authoritative Registry Component

This needs a separate enhancement - but kcp requires a separate
component `kcp-registry` that acts as an authoritative database that
shards can use to synchronize sensitive data.

The same restrictions should apply as for the cache-server, as in that
only kcp components are allowed to access it.

The concerns of the cache-server and registry are different in sofar
that the `cache-server` replicates shard-owned data to other shards. The
`kcp-registry` is a shared database from all shards.

### Authorization, Revocation and Rotating secrets

Whichever APIExport is named as the owner in the APIIdentity is the
authoritative source of the allowed writers to serve an API.

Revoking is replacing `spec.secretRef` or removing the respective entry from `spec.writers`.

Rotating is adding a new secret and updating `spec.secretRef` or
swapping the entry in `spec.writers`.

## Changes, Compatibility and Migration

1. Add `registry.kcp.io` and establish shared scope in cache-server.
2. Add `spec.secretRef`, `spec.identity.value`, etc.pp. and update APIExport controller to establish the APIIdentity ownership in the cache-server.
3. Update every component to reference `status.identity`, and refuse to act if it is empty.
4. Update controller to validate and confirmed write the APIExport to the cache-server.

That means what is currently used as the storage segment for existing
APIExports does not change and there is no migration or manual
intervention needed.

## Alternatives Considered

1. **Keeping the status quo** - the status quo cannot be kept as
   a leaked key requires migrating consumer storage.
2. **Making secrets obsolete** - While technically feasible it would
   make the APIExport origin the implicit identity, resulting in
   migrating an APIExport to another workspace or rehoming objects
   between APIBindings require migrating consumer storage.
3. **Introduce kcp-registry immediately** - the `kcp-registry` needs
   a separate enhancement and consideration with careful planning.

## Other considerations

### Identity squatting

As long as a party owns an identity they maintain control of it through
the APIIdentity mechanics. If a provider releases control of an
APIIdentity it is a concious decision by the provider.

Squatting outside of that can only happen when a provider deliberately
chooses guessable identities and does not pre-claim those identities.

A method to claim e.g. prefixes is not considered.

### Removing secrets alltogether

While technically possible it would be a very large change that needs
much more careful consideration.

API identities are used to identify exported APIs, even when e.g.
claiming another API from an APIExport: provider A exports API M,
provider B exports API Z, which also uses API M and claims some of its
resources).

To facilitate this API identities are required and must be publicly known.

The alternative would be to fix APIs to the logical cluster they
originate from - which would make serving an API from different
workspaces (e.g. to migrate) or to rehome an APIExport over time
impossible.
