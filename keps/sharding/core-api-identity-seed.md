# Core API Identities Without the Root Shard

## Summary

Core APIExport identities (`tenancy.kcp.io`, `topology.kcp.io`,
`shards.core.kcp.io`, `migration.kcp.io`, `cache.kcp.io`) are random keys
generated in `root` on the root shard's first boot. Every other component must
fetch the resulting hashes from root before its informers can start, so the
root shard is a hard startup dependency.

This KEP:
- derives core identity keys from a **private key** (for example, one issued
  by cert-manager) that is mounted into every shard from a Secret, via a new
  flag `--core-identities-key-file`,
- makes shards use the identities they **already have** (their `system:shard`
  APIBindings, or the cache), and the key only for identities that don't exist
  yet,
- tracks every shard's identities in one dedicated object, `CoreIdentitySet`,
  aggregated in `:admin`,
- makes the front-proxy read hashes from `:admin` instead of from root,
- keeps `--root-identities-file` (plain-text keys) **for development only**.

Hashes of existing installations never change. For now, core identities are
**immutable**. The configured key is version 1. A key change (version 2) is
detected but not acted on: shards keep serving the current identities. In the
future, a key change triggers identity rotation automatically
([KEP 0005](../core/0005-apiexport-identity-rotation.md)).

This is follow-up #1 of the [Admin workspace KEP](shard-aggregated-admin-view.md)
(epic [kcp-dev/kcp#4336](https://github.com/kcp-dev/kcp/issues/4336)).

## Motivation

Identity key → `root:kcp-system/<export>` Secret. Hash = `sha256(key)` →
`APIExport.status.identityHash`. Components inject the hash into wildcard URLs
(`/clusters/*/apis/tenancy.kcp.io/v1alpha1/workspaces:<hash>`). Without it,
the request returns NotFound. See the [Appendix](#appendix-identities-today)
for real objects.

| Component | Gets hashes from today | Blocks on root |
|---|---|---|
| root shard | its own APIExports | phase0 + APIExport controller |
| other shards | `system:shard` ConfigMap `apiexport-identity-cache`, else root | first boot, and `--root-shard-kubeconfig` is required |
| virtual-workspaces | its shard's ConfigMap, else root | first boot of its shard |
| front-proxy | root, via `--root-kubeconfig` | **every start**, before serving anything (even `/livez`) |

`--root-identities-file` can pre-create keys, but it keeps them as plain text
in a config file, and only the root shard reads it. We should deprecate this 
in the future and move towards core-identities flag to make identities static.

### Goals

- Shards resolve core identity hashes without the root shard.
- The front-proxy and virtual-workspaces start while root is down.
- No identity secret material in plain-text config. The input is a private
  key from a Secret, created and stored by standard certificate tooling.
- Existing installations upgrade with no hash change and no required action.
- Core identities are immutable for now. A changed hash is refused. A changed
  key is detected and reported, and will later trigger rotation.
- The key has a version, so that rotation can be added later.

### Non-Goals

- Moving the core APIExports out of `root` (bindings reference `root:<export>`).
- Rotating core identities (a future KEP, built on KEP 0005).
- Identities of non-core APIExports.
- Keys held in an HSM or other non-exportable store. The raw key bytes are
  required.

## Proposal

### 1. Identity key

New shard flag: `--core-identities-key-file=<path to PEM private key>`. It is
typically the `tls.key` of a cert-manager `Certificate` Secret, mounted into
**every shard**:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: kcp-core-identities
spec:
  secretName: kcp-core-identities
  commonName: kcp-core-identities
  issuerRef: {name: kcp-ca, kind: Issuer}   # any CA
  privateKey:
    algorithm: ECDSA
    size: 256
    rotationPolicy: Never   # the key is the identity; re-keying = rotation
```

Only the **private key** is used. The certificate and the CA contribute no
cryptography here. They provide standard tooling: the key is generated
in-cluster, stored as a Secret, mounted like every other kcp certificate, and
never typed into a config file. Certificate renewal is fine **as long as the
key stays the same**. cert-manager's default since v1.18 is
`rotationPolicy: Always`, which would change the key on renewal, so `Never` is
recommended. A changed key is detected and reported, but not used (§5). In the
future, it triggers rotation (§7).

It must be a **dedicated** key, shared by all shards. It must not be a
shard's serving or client key, because those differ per shard and rotate.

Derivation for core export `name`:

```
ikm   = PKCS#8 DER of the private key      # re-encoded, so PEM/PKCS#1/SEC1 give the same bytes
key   = hex(HKDF-SHA256(ikm, salt="kcp.io/core-apiexport-identity/v1", info=name, len=32))
hash  = hex(SHA256(key))                   # unchanged algorithm
keyID = hex(SHA256(SPKI DER of the public key))   # public; recorded and compared
```

Supported key types: ECDSA P-256/P-384, Ed25519, RSA ≥ 2048.

### 2. Version 1 = the current identities

- Every installation is at version 1, including existing ones without a key,
  whose random keys already live in `root`.
- The key **never replaces an identity that already exists**. It only fills
  gaps: every export on a fresh install, and new core exports added by a kcp
  upgrade.
- The first key a shard sees is recorded (`keyID`) as version 1.
- A different key is version 2. It is detected and reported, but not used
  (§5). In the future, it triggers rotation (§7).

Resolving a core export, in order:

| Component | Source (in order) |
|---|---|
| shard hosting `root` | existing `root:kcp-system` secret → `--root-identities-file` (dev) → the key |
| other shards | its own `system:shard` APIBindings (`boundResources[].schema.identityHash`) → the cache server's replicated root APIExports → the key → root (legacy, deprecated) |
| virtual-workspaces | its shard's `CoreIdentitySet` (§4) |
| front-proxy | `coreidentitysets` in `:admin`, via peers (`--root-kubeconfig` is no longer used for identities) |

- Only shards receive the key.
- An existing shard always has its `system:shard` APIBindings, so it restarts
  without root or the cache.
- A fresh shard resolves from the cache. With the cache down, it derives from
  the key and stays not-Ready until the verifier confirms the result. On an
  existing install, the key can't reproduce the old random keys.
- Front-proxy identity resolution becomes a readiness check. `/livez` no longer
  waits for it.

### 3. Root materialization

Before phase0, the root shard writes keys for exports that have **no secret
yet** into `root:kcp-system/<export>` (create-only). The APIExport controller
is unchanged.

### 4. `CoreIdentitySet`: the one place to look

Today, identity state is spread across root Secrets, root APIExport status,
the cache, per-shard APIBindings, and a per-shard ConfigMap, and none of them
says which one is authoritative. This KEP adds **one dedicated object per
shard** that records what the shard serves and why:

```yaml
apiVersion: core.kcp.io/v1alpha1
kind: CoreIdentitySet
metadata:
  name: cluster                 # singleton, in the shard's system:shard
status:
  shard: shard-1
  keyID: 5c3e...                # active (recorded) key; version 1
  observedKeyID: 5c3e...        # key currently mounted
  exports:
  - name: tenancy.kcp.io
    identityHash: 0e2fc432...   # what this shard serves
    source: Legacy              # Legacy | Key | DevFile
    keyID: ""                   # key the identity was derived from (empty for Legacy)
    acceptedHashes: [0e2fc432...]   # == [identityHash] today; old+new during a future rotation
  - name: topology.kcp.io
    identityHash: 9af66732...
    source: Legacy
  conditions:
  - type: Consistent            # matches root's APIExports (via cache) and peers
    status: "True"
  - type: KeyChanged            # observedKeyID != keyID (§5, rule 4)
    status: "False"
```

- **A system CRD** (`system-crds`), served only in `system:shard`. It doesn't
  depend on any APIExport identity, so it is readable before identities are
  resolved.
- **Written only by the shard itself.** Admission denies writes from everyone
  else. There is no `spec`: the inputs are the key and the existing
  identities, not user edits.
- **Replicated to the cache server**, like `Shard`. `:admin` serves
  `coreidentitysets` for all shards (named after the shard), so an admin sees
  the whole installation in one list:

  ```sh
  $ kubectl ws use :admin
  $ kubectl get coreidentitysets
  NAME      KEYID      OBSERVED   CONSISTENT   KEYCHANGED
  root      5c3e...    5c3e...    True         False
  shard-1   5c3e...    5c3e...    True         False
  shard-2   5c3e...    9d10...    True         True
  ```

- **It replaces:**
  - the `apiexport-identity-cache` ConfigMap, which is deprecated and still
    written for one release for older virtual-workspaces servers,
  - the identity conditions and annotations on `Shard` and on the root
    APIExports.
- **Where to look:**

  | Question | Answer |
  |---|---|
  | What does shard X serve, and why? | shard X's `CoreIdentitySet` (or `:admin`) |
  | Do all shards agree? | `kubectl get coreidentitysets` in `:admin` |
  | Which key is active or mounted? | `status.keyID` / `status.observedKeyID` |
  | Where is the identity *key* for an export? | `root:kcp-system/<export>`, needed only by the APIExport controller |

### 5. Immutability

The shard's `CoreIdentitySet` is its record. On startup, the shard checks the
key and the existing identities against it.

Rules:
1. A recorded export hash never changes.
2. New exports (added by kcp upgrades) may be recorded.
3. A `--root-identities-file` entry that disagrees with an existing identity is
   an error.
4. **Key change:** the mounted key's ID differs from the recorded active
   `keyID`. This happens, for example, when the certificate is re-keyed. It is
   **not fatal**. Serving needs only the recorded hashes, never the old key.
   The shard:
   - keeps serving its recorded identities (existing identities win),
   - sets `observedKeyID` and `KeyChanged=True` on its `CoreIdentitySet`,
   - uses **neither** key to create new identities while the change is
     pending, so a new core export waits.

   In this KEP, the only way to resolve it is to restore the previous key. In
   the future, a key change *is* the rotation trigger (§7).

On a violation of rules 1–3:
- **Known shard:** refuses to start its informers, and the error names the
  export and both hashes.
- **Root shard:** fails readiness. Existing secrets are never overwritten.
- **Fresh shard:** compares against the cache and its peers before it records
  anything. On a mismatch, it records nothing and fails readiness.
- **Front-proxy:** is not Ready while peers disagree.

The `identitycache` controller becomes the cross-shard verifier. It sets
`Consistent` on the `CoreIdentitySet`. Admission rejects
`APIExportIdentityRotation` for core exports.

### 6. `--root-identities-file`: development only

It is unchanged (plain-text keys, root shard only), and is documented as
dev/test only. Using it together with `--core-identities-key-file` logs a
warning. Production installations use the key.

### 7. Rotation later (not specified here)

**Rotation is triggered by a certificate key change.** No separate flag or
manual object is needed.

These are already in place:
- the `CoreIdentitySet` records the active `keyID`, the mounted
  `observedKeyID`, and `acceptedHashes` per export (§4, §5),
- serving continues on the recorded identities while a change is pending,
- the resolver model is `export → {active, accepted hashes}`.

A future KEP adds:
1. **Detection.** Every shard reports `observedKeyID` in its `CoreIdentitySet`.
2. **Consensus.** A controller on the shard hosting root acts only when
   **every** shard reports the same new `observedKeyID`. A half-rolled mount,
   or one misconfigured shard, never triggers anything. It stays a
   `KeyChanged` condition.
3. **Trigger.** The controller creates a `CoreIdentityRotation`
   (`spec.targetKeyID`) in `:admin`, one per target key, idempotently. For each
   core export, it creates the KEP 0005 `APIExportIdentityRotation` in root,
   with the new secret derived from the new key.
4. **Migration.** The KEP 0005 migrator drains storage on every shard. During
   the window, shards accept `{old, new}` hashes.
5. **Completion.** Each shard switches its `CoreIdentitySet` (`keyID`, hashes)
   to the new key, and the condition clears.

Guardrails for that KEP:
- A key change while a rotation is running is held until the rotation
  completes.
- A pause annotation on `CoreIdentityRotation` stops the rollout before
  `Migrating`.
- Adding a key to a legacy install (no recorded `keyID`) records it as version
  1 and triggers **nothing**. Moving legacy random identities onto the key is
  an explicit `CoreIdentityRotation`.
- With automatic triggering, cert-manager `rotationPolicy: Always` means a full
  core identity migration on every renewal. The recommendation stays `Never`,
  re-keying on purpose.

Open problem: KEP 0005's whole-workspace fence doesn't scale to
`tenancy.kcp.io`.

## Upgrade

| Case | Operator action | Result |
|---|---|---|
| Existing installation | none | Same hashes. Existing shards restart from their local APIBindings, new shards resolve via the cache, and the front-proxy uses `:admin` |
| Existing installation, with a key | create the `Certificate` and mount it into all shards | Same hashes, because existing identities win. The key covers only core exports added later |
| Fresh install | kcp-operator/Helm creates the `Certificate` | No root dependency from the first boot |
| Any installation | the key changes (for example, `rotationPolicy: Always`) | Keeps serving the recorded identities, reports `KeyChanged` on its `CoreIdentitySet`, and blocks new core exports until the key is restored. In the future, this triggers rotation |

## Changes

- New shard flag `--core-identities-key-file`.
- `--root-identities-file`: unchanged, documented as dev only.
- `--root-shard-kubeconfig`: optional for identities.
- `--root-kubeconfig` (front-proxy): no longer used for identities. It is
  removed at GA.
- A new system CRD `coreidentitysets.core.kcp.io`: a per-shard singleton in
  `system:shard`, replicated to the cache and served in `:admin`.
- The `apiexport-identity-cache` ConfigMap is deprecated.

## Implementation

1. `pkg/identity`: key loading (PEM → PKCS#8), `DeriveKey` (HKDF), and
   `KeyID`, with test vectors for each key type.
2. `pkg/server/bootstrap/identity.go`: the resolver chain from §2, and add
   `cache.kcp.io` to the export list.
3. `pkg/server/server.go`: the new flag, and write the missing keys into root.
4. The `CoreIdentitySet` system CRD, admission (only the shard writes it),
   cache replication, and startup checks. `identitycache` becomes the verifier.
5. `:admin` `coreidentitysets`. The front-proxy reads it and moves identity
   resolution to readiness. Virtual-workspaces read their shard's object.
6. kcp-operator/Helm: a `Certificate` with `rotationPolicy: Never`, mounted
   into all shards.
7. e2e:
   - shards and the front-proxy start with root down,
   - a changed key keeps serving the old identities, sets `KeyChanged`, and
     blocks new exports; restoring the key clears it,
   - `kubectl get coreidentitysets` in `:admin` lists every shard,
   - a key added to an existing install leaves all hashes unchanged,
   - a new core export is derived from the key,
   - the same key in PKCS#1 and PKCS#8 encodings gives identical hashes.

## Alternatives

- **A seed in `--root-identities-file`:** secret material in plain-text
  config. It is kept for dev only.
- **A raw random Secret (not a certificate):** works the same way, but has no
  standard tooling to generate and manage it. Any PEM key file is accepted, so
  this still works for operators who prefer it.
- **Only `:admin` or the cache, with no key:** fresh installations still
  depend on root generating the keys first.
- **Hard-coded identities:** anyone could mint the keys.
- **Moving the core exports to `system:shard`:** breaks `root:<export>`
  bindings. That belongs to the structural "hierarchy root" work.

## Risks

- **A leaked key** lets an attacker mint core identities. It is only mounted
  into shards, and it is as sensitive as the `root:kcp-system` secrets are
  today.
- **Key rotation by certificate tooling** (`rotationPolicy: Always`, a CA
  migration that re-keys). Shards keep serving and report `KeyChanged` (rule 4),
  but new core exports are blocked. With future automatic rotation, every
  re-key becomes a full migration. The docs and the operator recommend
  `rotationPolicy: Never`.
- **A lost key.** Existing identities keep working, because they already
  exist. Only new exports and fresh shards with the cache down are affected.
  Back up the Secret.
- **Immutability locks in a leaked key** until rotation exists. This is the
  same exposure as today.

## Open Questions

- Flag name: `--core-identities-key-file`?
- Should the operator refuse certificates without `rotationPolicy: Never`?
- Should a missing key be an error for multi-shard setups from Beta on?

## Appendix: identities today

From a local sharded setup (`export KUBECONFIG=.kcp/admin.kubeconfig`).
Shard-1 is reached directly with:

```sh
k1() { kubectl --kubeconfig .kcp-1/admin.kubeconfig --context shard-base \
         --server https://127.0.0.1:6445/clusters/"$@"; }
```

**Key: Secret in root**

```sh
$ kubectl get secret -n kcp-system tenancy.kcp.io -o jsonpath='{.data.key}'
<base64 random key, redacted>
```

**Hash: root APIExport**

```sh
$ kubectl get apiexports -o custom-columns=NAME:.metadata.name,HASH:.status.identityHash
cache.kcp.io         2bee60dc38d6b63748caa01b5b264aa96a45e5248d2d2a031390a095efcfc9e8
migration.kcp.io     2935c2d90707faf376db77b1f5eddbbcf15e9a0b70720a5422ca2cc349350be8
shards.core.kcp.io   be3fadca4186ee33347d2b95304d1953727541bc6ce60abc850d3d00af623052
tenancy.kcp.io       0e2fc43202bafe0e9b5d9b706617c66a5899e5bdbbd1a326e289b8e8bcfdf298
topology.kcp.io      9af667328c7b4b2e79ded87cd9d30713467e44d548f96f1ea1915f53f6e63f0f

$ echo -n "$(kubectl get secret -n kcp-system tenancy.kcp.io -o jsonpath='{.data.key}' | base64 -d)" | shasum -a 256
0e2fc43202bafe0e9b5d9b706617c66a5899e5bdbbd1a326e289b8e8bcfdf298
```

**Hash recorded on every shard: `system:shard` APIBinding**

```yaml
# k1 system:shard get apibinding tenancy.kcp.io -o yaml
status:
  boundResources:
  - resource: workspaces
    schema:
      identityHash: 0e2fc43202bafe0e9b5d9b706617c66a5899e5bdbbd1a326e289b8e8bcfdf298
```

**Hash cache on non-root shards: ConfigMap** (overwritten by `identitycache`)

```yaml
# k1 system:shard get cm -n default apiexport-identity-cache -o yaml
data:
  tenancy.kcp.io: 0e2fc43202bafe0e9b5d9b706617c66a5899e5bdbbd1a326e289b8e8bcfdf298
  ...
```

**The hash in use: wildcard request**

```sh
$ k1 '*' get --raw "/clusters/*/apis/tenancy.kcp.io/v1alpha1/workspaces"
Error from server (NotFound): the server could not find the requested resource
$ k1 '*' get --raw "/clusters/*/apis/tenancy.kcp.io/v1alpha1/workspaces:0e2fc432...f298"
{"apiVersion":"tenancy.kcp.io/v1alpha1","items":[{"kind":"Workspace", ...}]}
```

**Before / after**

| Object | Today | After |
|---|---|---|
| identity input | random, generated in root | a mounted private key (`--core-identities-key-file`); `--root-identities-file` is dev only |
| `root:kcp-system/<export>` | random key | unchanged if it exists; otherwise derived from the key |
| `APIExport.status.identityHash` | set by the controller | unchanged |
| `system:shard` APIBindings | hash stored, unused at startup | startup source |
| `apiexport-identity-cache` CM | copy of the cache, non-root only | deprecated |
| **`CoreIdentitySet`** (new) | – | **per-shard record: served hashes, source, key IDs, conditions; aggregated in `:admin`** |
| front-proxy | asks root | `:admin` `coreidentitysets` |
| wildcard URLs | `:<hash>` | unchanged (same hashes) |
