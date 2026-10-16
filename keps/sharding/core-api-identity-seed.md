# Core API Identities Without the Root Shard

## Summary

kcp's core APIs (`tenancy.kcp.io`, `topology.kcp.io`, `shards.core.kcp.io`,
`migration.kcp.io`, `cache.kcp.io`) each have an **identity**: a secret key
and a public hash derived from it. Today the root shard creates these keys
randomly on its first boot, and every other component has to ask the root
shard for the hashes before it can start. If the root shard is down, the rest
of the installation can't start either.

This KEP removes that dependency:

- Identities are derived from one **private key** that is shared by all
  shards (for example, a key issued by cert-manager), mounted from a Secret.
- Shards first use the identities they **already know**. The key is only used
  for identities that don't exist yet.
- Each shard publishes what it serves in a new object, `CoreIdentitySet`.
  Admins see all shards in one list in the `:admin` workspace.
- The front-proxy reads identities from `:admin` instead of the root shard.
- `--root-identities-file` (plain-text keys) stays for development only and
  is deprecated in favor of `--core-identities-key-file`.

Existing installations keep exactly the same identities. For now, core
identities **never change**. If the key changes, kcp notices and reports it,
but keeps using the current identities. Later, a key change will trigger an
automatic identity rotation
([KEP 0005](../core/0005-apiexport-identity-rotation.md)).

This is follow-up #1 of the [Admin workspace KEP](shard-aggregated-admin-view.md)
(epic [kcp-dev/kcp#4336](https://github.com/kcp-dev/kcp/issues/4336)).

## Motivation

An identity hash is part of every cross-workspace request for a core API, for
example `/clusters/*/apis/tenancy.kcp.io/v1alpha1/workspaces:<hash>`. Without
the right hash, the request fails. So every component needs the hashes before
it can do any work.

| Component | Where it gets the hashes today | Needs the root shard |
|---|---|---|
| root shard | creates them | – |
| other shards | a local copy, otherwise the root shard | on first boot |
| virtual-workspaces | its shard's local copy, otherwise the root shard | on its shard's first boot |
| front-proxy | the root shard | **on every start**, before it serves anything |

`--root-identities-file` can set the keys up front, but it stores them as
plain text in a config file, and only the root shard reads it.

### Goals

- Shards, the front-proxy and virtual-workspaces can start while the root
  shard is down.
- No identity secrets in plain-text config. The input is a private key stored
  in a Secret and managed with standard certificate tooling.
- Existing installations upgrade with no identity change and no required
  action.
- Core identities don't change for now. A changed key is detected and
  reported.
- The design leaves room for identity rotation later.

### Non-Goals

- Moving the core APIs out of the `root` workspace.
- Rotating core identities (a future KEP).
- Identities of non-core APIs.
- Keys that can't be exported, such as keys held in an HSM.

## Proposal

### 1. The identity key

A new shard flag, `--core-identities-key-file`, points to a private key file.
Usually this is the key of a cert-manager `Certificate`, mounted into **every
shard**:

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
    rotationPolicy: Never   # changing the key means changing identities
```

Only the private key matters. The certificate is just a convenient way to
create and store the key: it is generated in the cluster, kept in a Secret,
and mounted like any other kcp certificate. Renewing the certificate is fine
as long as the key stays the same. cert-manager's default is to create a new
key on renewal, so we recommend `rotationPolicy: Never`.

The key must be **dedicated** to this purpose and the same on all shards. It
must not be a shard's own serving or client key, because those differ per
shard and change over time.

Each core API's identity is derived from the key and the API's name, so every
shard computes the same identities from the same key. The format of the key
file (PKCS#1, PKCS#8, and so on) doesn't matter, only the key itself. kcp also
records a public **key ID** so it can tell whether the key has changed.
ECDSA, Ed25519 and RSA (2048 bits or more) keys are supported.

### 2. Existing identities

- The key **never replaces an identity that already exists**. It only fills
  in missing ones:replace all of them on a fresh install, and new core APIs added by
  a kcp upgrade.
- On an existing installation, the current random identities stay.
- The first key a shard sees is recorded as the active key.

Where each component gets its identities, in order of preference:

| Component | Sources |
|---|---|
| root shard | identities already stored in `root` → `--root-identities-file` (dev only) → the key |
| other shards | identities they already use locally → the cache server → the key → the root shard (legacy) |
| virtual-workspaces | its shard's `CoreIdentitySet` |
| front-proxy | `CoreIdentitySet`s in `:admin` |

- Only shards get the key.
- An existing shard always has its own identities, so it restarts without the
  root shard or the cache server.
- A new shard gets identities from the cache server. If the cache server is
  down, it derives them from the key and stays not-ready until they are
  confirmed to match the other shards.
- The front-proxy starts and reports itself live right away. It only becomes
  ready once it has the identities.

### 3. Filling in missing identities in root

When the root shard starts, it creates any missing core identity keys in
`root` from the key. It never overwrites existing ones.

### 4. `CoreIdentitySet`: one place to look

Today, identity information is spread across several places, and none of them
is clearly the source of truth. This KEP adds one object per shard that says
what the shard uses and why:

```yaml
apiVersion: admin.kcp.io/v1alpha1
kind: CoreIdentitySet
metadata:
  name: cluster
status:
  shard: shard-1
  keyID: 5c3e...                # the active key
  observedKeyID: 5c3e...        # the key currently mounted
  exports:
  - name: tenancy.kcp.io
    identityHash: 0e2fc432...   # what this shard uses
    source: Legacy              # Legacy | Key | DevFile
    keyID: ""                   # key the identity was derived from (empty for Legacy | DevFile)
    derivation: ""              # how it was derived (empty for Legacy | DevFile)
  - name: topology.kcp.io
    identityHash: 9af66732...
    source: Legacy
    keyID: ""
    derivation: ""
  - name: cache.kcp.io          # added by an upgrade, filled in from the key
    identityHash: c71d0a9e...
    source: Key
    keyID: 5c3e...
    derivation: hkdf-sha256/v1
  conditions:
  - type: Consistent            # matches the root shard and the other shards
    status: "True"
  - type: KeyChanged            # the mounted key differs from the active key
    status: "False"
```

- Every shard has one, and only that shard can write it. It has no `spec`.
- It is readable before the shard has resolved its identities.
- `keyID` and `derivation` record, per export, which key and which derivation
  algorithm and version produced the identity. This metadata lives **next to**
  the identity, not inside the hash: the hash stays a plain SHA-256 hex, the
  same format as every other APIExport identity, because it is part of request
  URLs (`workspaces:<hash>`), storage keys and labels, and legacy identities
  carry no prefix. A future derivation (`v2`) yields a different identity;
  these fields say which one is in use.
- All shards' objects are visible together in `:admin`:

  ```sh
  $ kubectl ws use :admin
  $ kubectl get coreidentitysets
  NAME      KEYID      OBSERVED   CONSISTENT   KEYCHANGED
  root      5c3e...    5c3e...    True         False
  shard-1   5c3e...    5c3e...    True         False
  shard-2   5c3e...    9d10...    True         True
  ```

- It replaces the `apiexport-identity-cache` ConfigMap, which is deprecated
  and kept for one more release for older virtual-workspaces servers.

### 5. Identities don't change

Each shard checks its `CoreIdentitySet` on startup. The rules:

1. An identity that is already in use never changes.
2. Identities for new core APIs (added by a kcp upgrade) can be added.
3. A `--root-identities-file` entry that disagrees with an existing identity
   is an error.
4. **The key changed** (for example, the certificate was re-keyed). Not
   fatal: the shard only needs the identities it already has. The shard:
   - keeps using its current identities,
   - reports `KeyChanged=True`,
   - doesn't create identities for new core APIs until the change is
     resolved.

   For now, the only fix is to restore the previous key. Later, a key change
   will start a rotation (§7).

If rules 1–3 are broken:
- an existing shard refuses to start, and the error names the API and both
  hashes,
- the root shard reports not-ready and never overwrites existing identities,
- a new shard compares with the other shards first and, on a mismatch,
  reports not-ready,
- the front-proxy is not ready while shards disagree.

kcp continuously checks that all shards agree and reports it in the
`Consistent` condition. Rotating a core identity by hand is rejected.

### 6. `--root-identities-file`: development only

The flag works as before, but is documented for development and tests only.
Using it together with `--core-identities-key-file` logs a warning. It is
deprecated and kept for one more release, so existing installations can
upgrade without changing their identities.

### 7. Future: rotation

Later, **changing the key will start a rotation** of the core identities.
This KEP already provides the inputs: the active and mounted key IDs, and
shards that keep working while a change is pending.

The rough idea for the future KEP:
1. Every shard reports the key it has mounted.
2. Nothing happens until **every** shard reports the same new key, so a
   half-finished rollout or one misconfigured shard can't trigger a rotation.
3. Then kcp rotates each core identity using
   [KEP 0005](../core/0005-apiexport-identity-rotation.md), accepting both
   the old and new identities while stored data is migrated.
4. When done, every shard switches to the new key.

Safeguards:
- A key change during a running rotation waits until it finishes.
- A rotation can be paused before migration starts.
- Adding a key to an existing installation only records it.
- With automatic rotation, `rotationPolicy: Always` would mean a full
  migration on every certificate renewal, so `Never` stays the
  recommendation.

Open problem: KEP 0005's migration approach doesn't yet scale to
`tenancy.kcp.io` as its global export! We will need to address this scalability issue in future work.

## Upgrade

| Case | What the operator does | Result |
|---|---|---|
| Existing installation | nothing | Same identities. Shards and the front-proxy no longer need the root shard to start |
| Existing installation, adding a key | create the `Certificate` and mount it into all shards | Same identities. The key is only used for core APIs added later |
| Fresh install | kcp-operator or Helm creates the `Certificate` | No root shard dependency from the start |
| The key changes | – | kcp keeps the current identities, reports `KeyChanged`, and blocks new core APIs until the key is restored |

## Changes

- New shard flag `--core-identities-key-file`.
- `--root-identities-file`: unchanged, development only. Deprecated for future removal.
- `--root-shard-kubeconfig`: no longer required for identities.
- `--root-kubeconfig` (front-proxy): no longer used for identities; removed
  at GA.
- New `CoreIdentitySet` resource: one per shard, all visible in `:admin`.
- The `apiexport-identity-cache` ConfigMap is deprecated.
- kcp-operator and Helm create the `Certificate` with `rotationPolicy: Never`.

## Testing

- Shards and the front-proxy start while the root shard is down.
- A changed key keeps the current identities, reports `KeyChanged`, and
  blocks new core APIs; restoring the key clears it.
- `kubectl get coreidentitysets` in `:admin` lists every shard.
- Adding a key to an existing installation changes no identities.
- A new core API gets its identity from the key.
- The same key in different file formats gives the same identities.

## Alternatives

- **Put a secret seed in `--root-identities-file`:** secrets in plain-text
  config. Kept for development only.
- **A plain random Secret instead of a certificate:** works, but lacks
  standard tooling. Any private key file is accepted.
- **Only read identities from `:admin` or the cache, without a key:** fresh
  installations would still need the root shard to create them first.
- **Hard-coded identities:** anyone could forge them.
- **Move the core APIs out of `root`:** breaks existing bindings to them;
  this belongs to separate work.

## Risks

- **A leaked key** lets an attacker forge core identities. It is only
  mounted into shards and is as sensitive as the identity keys in `root`
  are today.
- **Accidental re-keying** (for example `rotationPolicy: Always`). kcp keeps
  working but blocks new core APIs. With future automatic rotation, every
  re-key would mean a full migration. The docs and the operator recommend
  `rotationPolicy: Never`.
- **A lost key.** Existing identities keep working. Only new core APIs, and
  new shards while the cache server is down, are affected. Back up the
  Secret.
- **A leaked key can't be replaced** until rotation exists, as today.

## Open Questions

- Is `--core-identities-key-file` the right flag name?
- Should the operator refuse certificates without `rotationPolicy: Never`?
- From Beta on, should a missing key be an error for multi-shard setups?

## Appendix: before and after

| | Today | After |
|---|---|---|
| Identity source | random, created by the root shard | a shared private key (`--core-identities-key-file`); `--root-identities-file` for dev only |
| Existing identities | – | unchanged |
| Where to see identities | spread across several objects | `CoreIdentitySet`, all shards listed in `:admin` |
| Shard startup | needs the root shard on first boot | uses local identities, the cache, or the key |
| Front-proxy startup | needs the root shard every time | reads `:admin` |
| Request URLs | `...:<hash>` | unchanged (same hashes) |
