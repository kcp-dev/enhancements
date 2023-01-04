# Decouple workspace logical clusters from the workspace hierarchy

## Overview

Make workspace semantics self-contained and independent from parent workspaces.

## Motivation

Dependencies on the parent workspace make horizontal scaling hard. Either shards have to access the parent workspaces' shards during normal operation, or we have to replicate lots of parent state through-out the platform.

The following use-cases would benefit from self-contained workspaces:

1. checking logical cluster existence
2. authorizing workspace access
3. implementing a workspace sharing feature
4. feeding the front-proxy workspace index
5. virtual-workspaces serving per workspace
6. alternative workspace inter-relations than hierarchies (hierarchy should be a "battery included", but not at the core of kcp).
7. logical cluster authz and workspace authz would not differ.

This is a pre-req for adding shard support to kcp.

### Goals

1. allow shards to operate workspaces without having to know anything about parents, and with that no need to replicate high-cardinality workspace data.
2. change the user experience as little as possible.

### Non-Goal

1. To be 100% compatible with old APIs, especially `kcp.dev/cluster` annotation will change and cannot be used anymore for hierarchy traversal.
2. To keep `ClusterWorkspaces` accessible by external components. This has always been an internal API.

## Proposal

### Workspace existence and life-cycle

`Workspaces` become "real" objects, i.e. are not projected anymore. Every workspace in the hierarchy will be two parts: a `Workspace` in the parent, a singleton named `cluster` of kind `LogicalCluster` in `core.kcp.dev/v1alpha1` on a shard **inside** its logical cluster.

1. The user-experience using kubectl will not change:

   ```
   $ kubectl get workspaces
   NAME  PHASE   URL
   ws    Ready   https://.../clusters/gad78v6
   ```

   extended with

   ```
   $ kubectl get logicalclusters
   NAME     PHASE   URL
   cluster  Ready   https://.../clusters/gad78v6
   ```

1. A logical-cluster (an etcd prefix) is defined as a workspace through the existence of the `LogicalCluster` object. Authorization will use this to decide about existence of a workspace. This object is user-accessible, but read-only for non-system:masters. If we decide to need system data, we will introduce another object (in the `system:workspaces` logical cluster) that is 1:1 with `LogicalCluster`.

1. `phase` and `initializers` are stored on `LogicalCluster`, not in `Workspace` (anymore).

1. The logical cluster string is replaced with a base36 ID, i.e. `kcp.dev/cluster` will **not** be a workspace path anymore.

1. On deletion of a workspace:

   - the RBAC roles and role bindings are deleted last (delayed via a finalizer `core.kcp.io/logicalcluster-deletion` on the `LogicalCluster` object).
   - the finalizer `core.kcp.io/logicalcluster` on a `Workspace` of primary type will delay the `Workspace` object deletion until the actual logical cluster is deleted.
   - a controller on the logical-cluster-hosting shard will remove the finalizer of (a) and eventually of (b).
   - there can only be one primary `Workspace` object at a time. A by-workspace+name+UID reference to the `Workspace` on the `LogicalCluster` object is the single source of truth of who the primary is.

### Hierarchy

1. We store the workspace path in `kcp.dev/path` on the `LogicalCluster` object. This annotation and the hierarchy logic is optional.

1. A workspace `z` of ID `jk65lk3` under another workspace of path `x:y` of ID `gad78v6` can be accessed through the front-proxy via:

   1. `x:y:z`
   2. `gad78v6:z`
   3. `jk65lk3`.

   Access through the hosting shard is only guaranteed via `jk65lk3` (other paths might work depending on how much of hierarchy is stored on that same shard).

1. The front-proxy can add additional mappings from workspace paths to IDs with different, unique prefixes, e.g. `home-lukasz:subws`.

1. `Workspace` will become a link to `LogicalCluster` aka logical clusters. A `Workspace` is called `Primary` if its deletion will also delete the logical cluster content. A `Workspace` object of type `Symbolic` (to be added in the future) acts like a symlink in Unix.

1. The workspace hierarchy will become purely a virtual structure in-memory of the front-proxies, plus users can navigate through the tree
   keeping the path as state. From objects in a workspace, it is normally not possible to derive the workspace path directly. Some objects will carry the path via `kcp.dev/path` annotations.

1. The root workspace has ID "root".

1. There can be multiple ways to create a `LogicalCluster`:

   a) through a `Workspace` object
   b) through access to a user workspace UID (e.g. `base36(sha224(user:<userid>))`)
   c) through other objects, e.g. `WorkspaceRequest` in the future or in custom kcp installation.

1. Home workspaces are implemented through parent-less `LogicalCluster` objects. The bucket hiearchy will go away.

1. Object referencing other objects by workspace path require the `kcp.dev/path` annotation on those target objects.

1. Adding `kcp.dev/path` annotations is a non-privileged operation on object creation with empty value. The system will fill in the right value.

### Authorization

1. We add a `system:kcp:logical-cluster-admin` group that is allowed to create, update and delete `LogicalCluster`. This is used by the scheduler on the `Workspace` hosting shard to create a `LogicalCluster` object on the target shard.

1. We remove `workspaces/content` RBAC semantics and replace it with:

   a) access to a workspace is authorized through `verb=access`, `non-resource=/` inside of a workspace
   b) admin access is given through in-workspace binding of a `*/*` RBAC role
   c) deletion of a workspace is authorized through `verb=delete` on `logicalclusters` inside of a workspace **or** via the delete verb on the `Workspace` object of type `Primary`.

1. We remove filtering of workspaces by RBAC access. It is not anymore necessary to filter user home workspaces or organization workspaces because these become their own root of a hierarchy.

1. For top-level workspace authorization we will add an opt-in authorizer that checks group membership, driven by an annotation `authorization.kcp.io/required-groups` (with a comma-separated list of alternatives of a semicolon-separated list of required group names) on the `LogicalCluster` object. This is inherited to sub-workspaces on creation by copying over the annotation.

### Workspace Types

1. `ClusterWorkspaceType` is renamed to `WorkspaceType`.

1. Workspace types as an API stay in `tenancy.kcp.dev`. The concept of initializers is part of core and stored on `LogicalCluster` objects in the `core.kcp.dev/v1alpha1` API group.

1. Both phase and initializers are synced to the owning `Workspace` during the initialization phase.

1. The `WorkspaceType` virtual workspace apiserver serves `logicalclusters`, not `clusterworkspaces` anymore. Initializers must be updated by the initialization controllers on the former.

### Consequences

1. not every workspace has a parent
1. we have to distinguish between workspace references that are
   - `Workspace` names or paths
   - that workspace IDs.
1. we could have organization/tenants that are not prefixed with `root:`, e.g. `redhat:engineering:kcp`.

### kcp.io replaces kcp.dev

All occurrences of `kcp.dev` in the repository (e.g. in API group names, annotations and labels) are replaced with `kcp.io`.

## Implementation

## TODO/Question

## Alternatives

- instead of renaming `kcp.dev` to `kcp.io` we could have introduced `v1alpha2` APIs. But we used the chance to do the domain transition as these changes are breaking anyway.

## References

- drawing of Workspace-LogicalCluster relationship https://docs.google.com/drawings/d/198J6E5e3ZgJAf4aaVM4Swero5pXBgMKYgb-53i3dwAo/edit