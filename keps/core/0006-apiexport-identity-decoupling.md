# APIExport Identity Decoupling

Supersedes: [0005 APIExport Identity Rotation](0005-apiexport-identity-rotation.md).
Identities are never rotated and carry no secret, so the rotation object does not exist any
more. Kept from 0005: its storage migrator, now driving storage re-keying (§4).

Builds on: [0004 APIBinding Resource Adoption](0004-apibinding-resource-adoption.md). Its
zero-copy split keeps working through the legacy `secretRef` path during the deprecation
window (§7, §8). An explicit storage handover between exports is
[KEP 0007](0007-apiexport-handover-and-claim-references.md).

Status: draft. A reference implementation exists on the kcp pull request
`https://github.com/kcp-dev/kcp/pull/4484`. It also carries the parts split out into
[KEP 0007](0007-apiexport-handover-and-claim-references.md).

## Summary

Today an APIExport's identity is the hash of a secret. That one string does four jobs at
once: it is the etcd prefix where consumers' objects are stored, the `resource:identity`
suffix that routes wildcard requests, the public name that PermissionClaims refer to, and,
in theory, a proof of ownership. Because it is one string, the only answer to a leaked
secret is to change it, and changing it means moving every consumer's data and breaking
every claim that named the old value.

The ownership proof is also not real. kcp reads the secret once to compute the hash, writes
the hash into `status.identityHash`, and from then on everything trusts the status field,
which nothing guards. In reality the secret protects nothing.

This proposal makes the identity a plain identifier and separates the four jobs:

- the **identity** is an opaque, immutable string assigned once by kcp; no secret;
- the **storage prefix** is tracked per bound resource and may differ from the identity;
- claims keep naming `identityHash`; because identities never change, claims never break;
- moving data is a separate **storage re-keying** that never touches the identity;
- a registry (`APIIdentity`) answers "who holds identity X" installation-wide and lets core
  identities be found without the root shard;
- identities that are already shared by several exports are **converted once**, without
  moving data, at a moment the owner chooses.

Existing installations keep their identities and their etcd layout byte for byte.

## Motivation

- **Changing an identity is a data migration.** Every consumer workspace is rewritten,
  with an outage each. For `tenancy.kcp.io` that is every workspace in the installation.
- **Changing an identity breaks references.** Claims, cached resources and wildcard
  clients embed the hash.
- **Sharing an identity is done by copying a secret.** That is the only way 0004's
  zero-copy adoption can express "export B may serve what export A left behind". It is
  neither audited nor authorized, and nothing records which exports hold an identity.
- **Core identities need the root shard.** They are random values root creates on first
  boot; shards, the front proxy and virtual workspace servers must ask root before serving.
- **The status field is unguarded.** An admin of a provider workspace can write another
  export's hash into their own export's status and serve that export's consumers' data.

Once the identity is only an identifier, none of these remain: it never changes, so nothing
needs updating; there is no secret to leak or copy; core identities can be constants;
and a single admission rule closes the status hole.

### Goals

- The identity is immutable and public. Its value means nothing cryptographically.
- No secret is created for new APIExports or ClusterCachedResources.
- Every export has its own identity. Identities already shared by several exports are
  converted once (§7).
- Existing installations upgrade with no data movement.
- Shards, virtual workspace servers and the front proxy start without the root shard.
- `APIExportIdentityRotation` is removed. 0005's migrator is repurposed as storage re-keying.

### Non-goals

- Changing the etcd key layout or the `resource:identity` addressing scheme.
- Moving core APIs out of the root workspace.
- Forcing existing core identities onto well-known constants (optional, via re-keying).
- An explicit storage handover between exports, claims by export reference, and claim
  grants. See future [KEP 0007](0007-apiexport-handover-and-claim-references.md).

## Design

### 1. The identity is just an identifier

The identity of an export is an opaque, unique, immutable string that is also the default
storage prefix of its bound instances. Existing 64-hex values remain valid. It lives in a
new status field that mirrors `spec.identity`:

```yaml
# APIExport status with identity
status:
  identity:
    value: 2cynbfy2m0wtjqcs-8f1c0a7e...
    source: Generated        # Generated | Pinned | Legacy (from a secret) | WellKnown | Converted (§7)
  identityHash: 2cynbfy2m0wtjqcs-8f1c0a7e...   # deprecated: same value as identity.value
```

`status.identityHash` is kept and always written with the same value, because it is read
by kcp's own controllers and by external tooling (kcp-operator, api-syncagent, platform
operators). It goes away at N+3 with `secretRef`. The name `identityHash` on
`PermissionClaim`, `boundResources[].schema` and `ClusterCachedResource` is unchanged; it
refers to the same value and renaming it is a matter for the next API version.

New identities are **owner-prefixed**: `<logical-cluster-name>-<random>`. Admission can
check that an identity pinned on an export in cluster `c` starts with `c-` without any
lookup. Core exports on fresh installations get **well-known** identities such as
`system-tenancy.kcp.io`; no user cluster is named `system`, so nothing can collide.
Upgraded installations keep their random core identities.

### 2. Storage is per resource

Objects live in etcd at `/registry/<group>/<resource>/<prefix>/<workspace>/...`. Today the
prefix is the identity, implicitly: the bound CRD on the shard carries
`boundResources[].schema.identityHash` as an annotation and the storage layer builds the
prefix from it. Prefix and identity cannot differ.

This proposal records the prefix explicitly, as a new `storage.key` on each
`boundResources[]` entry of the consumer's `APIBinding`, and the storage layer reads the
prefix from there instead of from the identity. In the steady state the two are equal and
nothing observable changes. They differ in exactly two situations: after a legacy
conversion (§7), and during a re-keying (§4). This is the one place identity and storage
are decoupled; nothing in `spec` puts an export on another export's prefix.

```yaml
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport
metadata:
  name: networking
spec:
  identity:
    # Pin an identity instead of letting kcp generate one. Must start with this
    # export's logical cluster name and "-", and must not be held by anyone.
    # Honored only while status.identity is empty. For GitOps recreate.
    value: 2cynbfy2m0wtjqcs-8f1c0a7e...
    # Deprecated. Read once for an export without identity; never created by kcp.
    secretRef: {namespace: kcp-system, name: networking}
  resources:
  - name: vnets
    group: networking.io
    schema: v250210-8c1e4f2.vnets.networking.io
    storage: {crd: {}}
status:
  identity: {value: 2cynbfy2m0wtjqcs-8f1c0a7e..., source: Generated}
  identityHash: 2cynbfy2m0wtjqcs-8f1c0a7e...   # deprecated mirror
  storage:                         # prefixes bindings of each resource use
    resources:
    - {group: networking.io, resource: vnets, keys: [2cynbfy2m0wtjqcs-8f1c0a7e...]}
  conditions:
  - type: IdentityValid
  - type: StorageShared            # some binding of this export stores under another
                                   # export's prefix, or vice versa (§7)
```

Rules:

- **The status rule.** `status` of an APIExport is written only by kcp. Writes by anyone
  else are rejected. This closes the hole described in Motivation and lands first (§8).
- `status.identity.value` is immutable once set. The one exception is the legacy
  conversion (§7), performed by kcp.
- `spec.identity.value` must carry the owner prefix, must not be held by another export or
  recorded in the registry for another owner, and is immutable once the identity is set.

**Splitting or replacing an export** (0004) during the deprecation window works as today:
the new export names the old export's `secretRef`, is hashed once, and shares the
identity. §7 then converts the pair. After `secretRef` is gone, a split is a re-keying of
the affected resources onto the new export (§4). A zero-copy handover with distinct
identities is [KEP 0007](0007-apiexport-handover-and-claim-references.md).

### 3. Claims are unchanged

A claim names the claimed export's `identityHash`, as today. The claimer still looks the
value up; what changes is that the value never changes afterwards, so a claim never breaks
and nothing ever has to edit one. The one exception is a legacy shared identity converted
under §7, at the owner's request: claims that named the old hash for a converted holder's
resources stop matching it and must be updated, exactly as after an identity change today.

**Knowing an identity is not consent.** The identity is public. A claim is a claim on a
stranger's resource and needs the consumers' acceptance plus the claimed export's maximal
permission policy, exactly as today. Naming the export instead of its hash, and letting the
claimed export grant claims up front, are in [KEP 0007](0007-apiexport-handover-and-claim-references.md).
Claims without any hash, gated on the claimed side, are the subject of
[Identity-Agnostic Permission Claims](https://github.com/kcp-dev/enhancements/pull/22).

### 4. Re-keying moves data, never the identity

0005's migrator drains a consumer workspace's objects from one prefix to another. It is
kept, but it moves storage only:

```yaml
apiVersion: migration.kcp.io/v1alpha1
kind: APIExportStorageRekey            # replaces APIExportIdentityRotation
metadata:
  name: vnets-onto-own-prefix
spec:
  export: {path: root:org:networking, name: networking-v2}   # a converted holder (§7)
  resources: [{group: networking.io, resource: vnets}]   # omitted: every CRD resource
  targetKey: ""                        # omitted: the export's identity; other values are
                                       # for platform administrators (e.g. system-tenancy.kcp.io)
status:
  phase: Pending | Draining | Completed | Failed
  totalBindings: 120
  migratedBindings: 87
  shards: [{shard: root, totalBindings: 80, migratedBindings: 80, storageKey: ...}]
```

The controller marks the export with the `migration.kcp.io/active-rekey` annotation, which
replicates to every shard. Each shard's binding reconciler records the target as a
`storage.pendingKeys` entry on the affected bindings; each shard's migrator fences the
workspace, copies the objects, flips `storage.key`, deletes the old prefix and reports its
counts into `status.shards`. The re-keying completes when every shard reports all its
bindings drained. On a shard, a schema is served by one bound CRD per storage key, so
bindings on the old and the new prefix coexist during the drain.

Re-keying needs no per-object bookkeeping. A workspace has exactly one APIBinding per
group/resource, so every object under a prefix belongs to the export that workspace is bound
to. Moving "B's vnets out of A's prefix" means visiting the workspaces bound to B, which is
what the migrator already does.

Uses: carrying a converted export's data onto its own prefix (§7), splitting an export
once `secretRef` is gone (§2), and optionally moving core exports onto their well-known
identities. Authorization follows
0005: the type is served by the platform-owned `migration.kcp.io` export, so binding it is
the capability; one active re-keying per export and a cooldown.

### 5. The registry

Every identity is listed as an `APIIdentity` in the Admin workspace, named after the
identity. It is derived from the APIExports by the shard hosting the owner and replicated
through the cache server. Only `spec` is writable; it holds the knobs of the legacy
conversion (§7).

```yaml
apiVersion: admin.kcp.io/v1alpha1
kind: APIIdentity
metadata:
  name: 2cynbfy2m0wtjqcs-8f1c0a7e...
spec:                                    # knobs of the legacy conversion (§7)
  convert: true
  owner: {cluster: 2cynbfy2m0wtjqcs, export: networking}
status:
  owner: {cluster: 2cynbfy2m0wtjqcs, export: networking}
  groupResources: [{group: networking.io, resource: vnets}]
  legacyHolders: []                      # exports that shared this identity before conversion
  storageSharedWith:                     # other exports whose bindings still store
  - export: {cluster: 2cynbfy2m0wtjqcs, export: networking-v2}   # under this prefix (§7)
    resources: [{group: networking.io, resource: vnets}]
  core: false
  conditions:
  - type: StorageShared
  - type: Orphaned                       # owner deleted, data may remain
  - type: LegacyShared                   # several exports hold it (§7)
```

The object outlives its export. When the owner is deleted it stays `Orphaned` until no
binding stores under the identity, so no other export can pin it meanwhile; the owner
cluster may reclaim it (GitOps delete and recreate).

### 6. Core identities without the root shard

Fresh installations give core exports well-known identities, so there is nothing to look
up. Upgraded installations keep their random values; the `APIIdentity` registry lists them
installation-wide through the cache server, with the root workspace's APIExports and then
the root shard as legacy fallbacks. Operators may move existing core identities onto the
constants with a re-keying, one export at a time.

### 7. Existing shared identities

Exports that only **claim** other exports' resources need nothing: identities do not
change, so claims keep matching.

Exports that **share one identity** (a 0004 split, or a copied secret) break the "one
identity per export" rule on upgrade day. They are converted once, without moving data:

1. **Grandfathering.** The registry records the identity as `LegacyShared` with its
   `legacyHolders`; each holder gets `IdentityLegacyShared: True`. Nothing else changes.
   New holders may still join through `secretRef` until N+2 (§8) and are converted like
   the rest. An installation may stay here indefinitely (`spec.convert: false`).
2. **Conversion**, on request (`spec.convert: true`). The owner (`spec.owner`, by default
   the oldest holder) keeps the identity. Every other holder gets a fresh owner-prefixed
   identity (`identity.source: Converted`) and keeps its bindings on the old prefix:
   `storage.key` is unchanged, so no data moves and no fence is needed; kcp rewrites
   `boundResources[].schema.identityHash` on those bindings to the new value.
3. **Stale claims.** A claim that named the old hash for a resource only a converted
   holder serves no longer matches anything and is flagged `ClaimStale` on the claiming
   export. Its owner updates the hash and consumers re-accept; both are ordinary edits,
   the same as after any identity change today. The owner of the shared identity picks
   the moment and can coordinate with the claimers first.
4. **Optional re-keying** carries the converted export's data onto its own prefix.

A shared identity is never converted automatically. An installation that never sets
`spec.convert` keeps its shared identities as they are.

| Release | What happens |
|---|---|
| N | gate alpha; grandfathering and on-request conversion with the gate on; `status.identity` written, `status.identityHash` mirrors it; `secretRef` deprecated |
| N+1 | gate on by default |
| N+2 | last release honoring `secretRef` |
| N+3 | `secretRef`, `status.identityHash` and the `apiexport-identity-cache` ConfigMap removed |

### 8. Feature gate

Everything but the status immutability rule sits behind the alpha gate
`APIIdentityDecoupling`. With the gate off, kcp keeps creating and reading identity
secrets. With it on: new exports get generated identities and no secret; existing exports
keep theirs (`identity.source: Legacy`) and their secret is never read again; an export with an
empty status and a `secretRef` is hashed from the secret one last time, so GitOps flows that
pre-create secrets still land where they expect during the deprecation window.

The status rule is not gated. It fixes a current defect and lands first.

## API changes

- `Identity` gains `value`; `secretRef` is deprecated.
- `APIExportStatus` gains `identity: {value, source}`; `identityHash` is deprecated and
  mirrors `identity.value` until N+3. It also gains `storage: {resources: [{group,
  resource, keys}]}` and the conditions `StorageShared`, `ClaimStale`,
  `IdentityLegacyShared`.
- `BoundAPIResource` gains `storage: {key, pendingKeys}`; 0005's `identityHashes` is
  removed. APIBinding gains the `StorageRekeyCompleted` condition.
- New: `migration.kcp.io/v1alpha1 APIExportStorageRekey`, `admin.kcp.io/v1alpha1
  APIIdentity`. Removed: `APIExportIdentityRotation` (unreleased).

## Upgrade

| Case | Operator action | Result |
|---|---|---|
| Existing installation | none | same identities and etcd layout; secrets stop being read |
| Enabling the gate | `--feature-gates=APIIdentityDecoupling=true` | new exports have no secret; old untouched |
| Fresh installation | none | core exports use well-known identities |
| Export recreated by GitOps | set `spec.identity.value` to the old identity | consumers' data preserved |
| Export split (0004) | during the window: target names the source's `secretRef`; after N+2: re-key onto the target | converted without data movement, or data carried over |
| Pre-created identity secrets | keep during the window, then switch to `value` | identity unchanged |
| Exports sharing one identity | none to keep sharing; `APIIdentity.spec.convert: true` (and `spec.owner`) to separate | converted without data movement; claims naming the old hash for a converted holder are flagged and need an edit |

## Security

Knowing an identity gives an attacker nothing. Claiming it needs consumer acceptance and
passes the claimed export's maximal permission policy. Scanning its prefix needs system
credentials. Serving under it, the only real attack, is blocked by the status rule, the
owner-prefix check on pins, and the registry keeping held and orphaned identities
unpinnable. Today the serving attack is open twice: with a leaked secret, and with no
secret at all through the unguarded status. This proposal removes the secret and guards
the status.

A converted legacy pair shares storage until re-keyed, visible as `StorageShared` on both
sides and in the registry. While it lasts, each holder's virtual workspace sees the other's
customers' objects for that resource over wildcard reads, which is no worse than today for
a pair that already shared a secret. Re-keying ends it. Filtering wildcard reads by
"workspace is bound to this export" is independent of identity and left to its own
proposal. Re-keying keeps 0005's protections against abuse: platform-owned export, one
active operation per export, cooldown.

## Alternatives considered

| | Status quo + 0005 | This proposal | Fingerprint registry | Deterministic identity |
|---|---|---|---|---|
| Identity changes on rotation | yes, data moves | no secret to rotate | no | no secret |
| Claims break on rotation | yes | never | never | never |
| Secrets to manage | one per export | none | one or more per export | none or one seed |
| Ownership model | possession, unenforced | admission plus owner prefix | allow-listed key fingerprints | derived from owner |
| Core identities without root | no | registry plus constants | registry | derived |

- **Fingerprint registry.** Keep secrets but separate them from the storage key; the
  cache server records which key fingerprints may serve which prefix. It fixes rotation
  but keeps secrets, their sprawl and their leak procedure, and in kcp no client ever
  presents the key, so "validate at serving time" collapses into the same admission check
  this proposal does with object references. Kept as a possible extension if portable
  ownership proof is ever needed.
- **Deterministic identities** (hash of cluster and name, or of UID). Removes the secret
  and the root dependency, but a UID-based value does not survive recreate. The owner
  prefix keeps the useful property without binding the identity to a derivation.
- **Status quo.** Commits to per-rotation migrations with outages, alias code in four
  components, and secrets whose only function is to be hashed once.
