# Logical Cluster Migration

## Overview

This proposal drafts a mechanism to migrate logical clusters between shards.

## Motivation

Shards grow with time, as do the logical clusters they contain.

Currently a kcp instance can scale horizontally by adding more shards or
vertically by increasing the available resources for a shard.

While increasing the resources for a shard is an option there is an
upper limit as to what the underlying infrastructure can provide.

Adding more shards is only a solution for new logical clusters. Not if
a shard is already overprovisioned. Similarly if the existing logical
clusters on a shard grow over time the total size of the logical
clusters might start to reach etcd limits.

### Goals

1. Provide a primitive for platform admins to migrate logical clusters between shards
2. Logical clusters must retain their identity
3. The migration must not interrupt other operations
4. The migration may interrupt operations related to the logical cluster

### Non-Goals

1. Automatic shard rebalancing

## Proposal

Add a new per-shard system workspace `system:migration` to contain shard-local migration objects.

Introduce the API group `migration.kcp.io` with the type `LogicalClusterMigration`:

```yaml
kind: LogicalClusterMigration
apiVersion: migration.kcp.io/v1alpha1
metadata:
  name: migration
spec:
  logicalCluster: 1a2b3c4d
  destinationShard: shard-b
status:
  phase: Migrating
  conditions:
  - lastTransitionTime: "2026-05-01T22:28:40Z"
    status: "True"
    type: OriginPreparing
  - lastTransitionTime: "2026-05-01T22:29:00Z"
    status: "True"
    type: OriginReady
  - lastTransitionTime: "2026-05-01T22:30:15Z"
    status: "True"
    type: MigrationStarted
```

This object exists in triplicate:

1. The version the admin created
   This can be anywhere the admin bound the API.
2. In the `system:migration` workspace of the origin shard
3. In the `system:migration` workspace of the destination shard

The resource is replicated to the cache-server, which is used to
coordinate the migration.

The stati and phase from the copies from the origin and destination
shard are reflected on the admin's copy.

The origin and destination are only updating their own copy in their own
workspace and reacting to updates on the other shard's copy.

### migration virtual workspace

A migration virtual workspace deployed as part of the virtual workspaces. Similar to the initializing/terminating workspaces and acts as a content proxy for other shards to the logical cluster currently in migration.

### Preventing reconciles during migration

Reconciles must be prevented from two access points - the user and kcp itself.

#### Users

The users (including admins) will be blocked with an annotation `internal.kcp.io/migrating` similar to the existing `internal.kcp.io/inactive` on the logical cluster during the migration.

Clients connecting through the front-proxy will only see the cluster not being available during the migration but otherwise see no difference.

multicluster-runtime clients using kcp's multicluster-provider will be able to handle the migration - the recent refactor already decoupled clusters from their shards in the internal storage. The provider will just need adjustments for the cache as clusters on a shard currently share the wildcard cache.

Other clients that e.g. connect directly to the shard or have their own handling for virtual workspaces will break regardless of what we do. Though something to try here would be a 301 permanent redirect.

#### kcp

kcp itself will filter objects related to this logical cluster on the informer level and purge the stores.

> [!NOTE]
> See the proof-of-concept PR on this: https://github.com/kcp-dev/kcp/pull/4124

### Process

The entire process is orchestrated through the `LogicalClusterMigration` objects, meaning shards communicate via the cache-server.

1. The origin shard annotates the logical cluster with `internal.kcp.io/migrating=true`.

   The annotation is similar to `internal.kcp.io/inactive` in that it
   prevents access. The important distinction is that the `inactive` still
   allows admins access - the `migrating` annotation does not.

   This also configures the front-proxy to prevent access.

2. The origin shard configures its informers to ignore objects related to this LC.

3. The origin shard updates the status of their copy of the `LogicalClusterMigration` object with the relevant vw information.

4. The destination shard configures its informers to ignore objects related to the migrating LC to prevent e.g. the regular resync from discovering objects in etcd and starting reconciles on the incomplete logical cluster.

5. The destination shard reads the objects through the migrating virtual workspace (retrieved from the `LogicalClusterMigration` object from the origin) and writes them directly to the etcd.

   After the data is copied and verified this is reflected on the destination copy of the `LogicalClusterMigration` object.

6. The origin shard deletes logical cluster object and its objects from the etcd.

   At this point the origin shard has no association with the logical cluster anymore.

7. The destination shard creates the logical cluster without the migrating annotation.

   This causes the front-proxy to pick up the new location of the LC.

8. Trigger resync/relist of informers on the destination shard

   Resyncing the informers should cause e.g. API sharing mechanisms to be updated.

#### Data completeness

The destination shard is responsible for validating that the copied data
is complete and equivalent to the data on the origin shard.

Something to explore would be a `SelfAccessReview` or
`TokenReview`-style resource in the `migration.kcp.io` API that allows
to request a merkle tree hash of the objects to compare against the
local copy.

#### Failstate Handling

If the process stalls or fails at any point the error will be noted on
the `LogicalClusterMigration` object and reconciliation will halt to
prevent data loss.

At that point an admin will have to take action.

By copying over all objects first it is guaranteed that the entire
logical cluster is always available on either the origin or destination
shard and allows recovery.
