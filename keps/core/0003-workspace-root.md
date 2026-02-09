# Workspace Root (Forest Support)

## Summary

Introduce a `WorkspaceRoot` API object to enable forest-type workspace hierarchies in kcp,
allowing multiple independent workspace trees to coexist. This addresses security concerns
with the current single-root structure where organizational hierarchy is exposed to all users,
and provides better tenant isolation by giving each organization its own opaque root identifier.

Currently, users navigating kcp workspaces can infer organizational structure:
```
root:orgs:company-a:team1:project
root:orgs:company-b:team2:project
```

With WorkspaceRoot, each organization gets its own isolated tree with an opaque identifier:
```
a1b2c3d4:team1:project  # company-a's tree
x9y8z7w6:team2:project  # company-b's tree
```

## Motivation

### Current State Problems

1. **Hierarchy Exposure**: The current single-root structure (`root:orgs:...`) exposes the
   organizational hierarchy to all users. Even without access, users can guess names, based
   on naming conventions.

2. **Predictable Paths**: Organization names in workspace paths are predictable. Knowing one
   organization exists (e.g., `root:orgs:acme`) allows guessing siblings (`root:orgs:contoso`).
   This is similar to how knowing one AWS account ID allows guessing others.

3. **Internal Structure Leakage**: The `root:orgs` path structure is an internal system mechanic
   that exposes implementation details to consumers. Organizations should not need to know they
   exist under a shared `root:orgs` prefix.

4. **Navigation Leakage**: Users can walk up the hierarchy using `ws use ..` and discover
   sibling workspaces, even if they receive permission denied errors. While this can be mitigated with rbac,
   but its very easy to misconfigure and leave gaps.
   ```bash
   $ kubectl ws use :root:orgs:bob
   Error: the server has asked for the client to provide credentials  # 401 - exists!

   $ kubectl ws use :root:orgs:bob1
   Error: access to workspace "root:orgs:bob1" denied  # 403 - doesn't exist
   ```

### Goals

1. Enable creation of independent workspace trees (forest structure) with opaque root identifiers.
2. Provide tenant isolation where organizations cannot infer existence of sibling organizations.
3. Maintain compatibility with existing workspace hierarchy mechanics within each tree.
4. Support WorkspaceType inheritance and WorkspaceAuthenticationConfiguration within trees.
5. Keep the API simple and focused on tree orchestration.

### Non-Goals

1. Advanced scheduling across shards (basic scheduling only).
2. Replacing the existing workspace hierarchy model - this extends it.
3. Cross-tree workspace references or mounts (each tree is isolated). This is out of scope for this KEP but 
   technically possible in the future if needed.
4. Automatic migration of existing workspaces to new trees.
5. Facade/translation layer - these are real separate roots, not path rewriting.

## Proposal

### WorkspaceRoot API

Introduce a new API object `WorkspaceRoot` in the `tenancy.kcp.io` API group:

```go
// WorkspaceRoot defines an independent workspace tree root.
// Creating a WorkspaceRoot provisions a new LogicalCluster that serves
// as the root of an isolated workspace hierarchy.
type WorkspaceRoot struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   WorkspaceRootSpec   `json:"spec,omitempty"`
    Status WorkspaceRootStatus `json:"status,omitempty"`
}

type WorkspaceRootSpec struct {
    // Name is the optional human-readable name for this root.
    // If not provided, a random base36 identifier will be generated.
    // +optional
    Name string `json:"name,omitempty"`

    // Type references a WorkspaceType that defines the initial configuration
    // for the root workspace, including allowed child types.
    // +optional
    Type *WorkspaceTypeReference `json:"type,omitempty"`
}

type WorkspaceRootStatus struct {
    // Phase indicates the current state of the workspace root.
    // +kubebuilder:default=Scheduling
    Phase WorkspaceRootPhase `json:"phase,omitempty"`

    // RootPath is the path identifier for this workspace tree root.
    // This is the opaque identifier users will use to access this tree.
    // Example: "a1b2c3d4" rather than "root:orgs:company-a"
    RootPath string `json:"rootPath,omitempty"`

    // URL is the base URL for accessing workspaces in this tree.
    // +optional
    URL string `json:"url,omitempty"`

    // Conditions represent the current state of the workspace root.
    // +optional
    Conditions conditionsv1alpha1.Conditions `json:"conditions,omitempty"`

    // LogicalCluster is the name of the LogicalCluster backing this root.
    // +optional
    LogicalCluster string `json:"logicalCluster,omitempty"`
}

type WorkspaceRootPhase string

const (
    WorkspaceRootPhaseScheduling   WorkspaceRootPhase = "Scheduling"
    WorkspaceRootPhaseInitializing WorkspaceRootPhase = "Initializing"
    WorkspaceRootPhaseReady        WorkspaceRootPhase = "Ready"
    WorkspaceRootPhaseDeleting     WorkspaceRootPhase = "Deleting"
)
```

### Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         kcp cluster                             │
├─────────────────────────────────────────────────────────────────┤
│  root (system workspace)                                        │
│  ├── orgs:workspaceroots (stores WorkspaceRoot objects)         │
│  │   ├── WorkspaceRoot/company-a → provisions "a1b2c3d4"        │
│  │   ├── WorkspaceRoot/company-b → provisions "x9y8z7w6"        │
│  │   └── WorkspaceRoot/company-c → provisions "m5n6o7p8"        │
│  └── ...                                                        │
├─────────────────────────────────────────────────────────────────┤
│  a1b2c3d4 (company-a's root)     ← independent tree             │
│  ├── team1                                                      │
│  │   ├── project1                                               │
│  │   └── project2                                               │
│  └── team2                                                      │
├─────────────────────────────────────────────────────────────────┤
│  x9y8z7w6 (company-b's root)     ← independent tree             │
│  ├── engineering                                                │
│  └── sales                                                      │
├─────────────────────────────────────────────────────────────────┤
│  m5n6o7p8 (company-c's root)     ← independent tree             │
│  └── ...                                                        │
└─────────────────────────────────────────────────────────────────┘
```

### User Experience

Users interact with their workspace tree using the opaque root identifier:

```bash
# User from company-a sees:
$ kubectl ws .
Current workspace is 'a1b2c3d4:team1:project1'.

$ kubectl ws tree
a1b2c3d4
├── team1
│   ├── project1 (current)
│   └── project2
└── team2

# Navigation works within the tree
$ kubectl ws use :a1b2c3d4:team2
Current workspace is 'a1b2c3d4:team2'.

# Cannot navigate to sibling trees (no visibility, must have access to know the root path)
$ kubectl ws use :x9y8z7w6
Error: access to workspace "x9y8z7w6" denied

# Kubeconfig references use opaque paths
$ kubectl config view
clusters:
- cluster:
    server: https://kcp.example.com/clusters/a1b2c3d4:team1:project1
  name: workspace
```

### Controller Behavior

The WorkspaceRoot controller is responsible for:

1. **Scheduling**: Selecting a shard for the new root LogicalCluster.
2. **Provisioning**: Creating the LogicalCluster with a unique base36 identifier.
3. **Initialization**: Applying WorkspaceType configuration if specified.
4. **Lifecycle Management**: Handling deletion with proper cleanup.

```
WorkspaceRoot Created
        │
        ▼
┌───────────────┐
│  Scheduling   │ ─── Select target shard
└───────┬───────┘
        │
        ▼
┌───────────────┐
│ Initializing  │ ─── Create LogicalCluster, apply type config
└───────┬───────┘
        │
        ▼
┌───────────────┐
│    Ready      │ ─── Tree is accessible
└───────────────┘
```

### Deletion Behavior

When a WorkspaceRoot is deleted:

1. A finalizer prevents immediate deletion.
2. All child workspaces within the tree are deleted recursively.
3. The root LogicalCluster is deleted.
4. The finalizer is removed and the WorkspaceRoot object is deleted.

This cascading deletion is guarded by finalizers on child workspaces to ensure
proper cleanup order.

### API Location and RBAC

The WorkspaceRoot API is made available through APIExport/APIBinding:

1. The `tenancy` APIExport in root workspace includes WorkspaceRoot.
2. Administrators bind the tenancy API to workspaces where they want to allow
   tree creation.
3. No binding = no permission to create WorkspaceRoots.

This follows the existing pattern for Workspace API availability and provides
natural RBAC scoping.

### Naming and Identifiers

- **User-provided names**: If `spec.name` is provided, it's used as metadata but
  the root path is still a generated opaque identifier.
- **Generated identifiers**: Base36 encoded, providing sufficient entropy to
  prevent guessing (e.g., `a1b2c3d4`, `x9y8z7w6`).
- **Root workspace**: The system `root` workspace continues to exist for system
  resources but organizations get their own trees.

## Implementation Phases

### Phase 1: Core API and Controller

1. Define WorkspaceRoot CRD in `tenancy.kcp.io/v1alpha1`.
2. Implement basic controller for lifecycle management.
3. Basic scheduling (single shard initially).
4. Integration with existing LogicalCluster provisioning.

### Phase 2: WorkspaceType Integration

1. Support WorkspaceType reference in WorkspaceRoot spec.
2. Apply type configuration to root workspace.
3. Inherit WorkspaceAuthenticationConfiguration.

### Phase 3: Multi-Shard Support

1. Shard selection for WorkspaceRoot placement.
2. Cross-shard tree management.

## Alternatives Considered

### Alternative 1: Modify Workspace API with `--root` flag

```bash
kubectl ws create bar --enter --root
# Creates :bar as new root instead of :root:current:bar
```

**Rejected because:**
- Changes workspace behavior based on flags, making the API inconsistent.
- Breaks hierarchical semantics (`ws use ..` becomes ambiguous).
- Cannot scope permissions - anyone with Workspace create can create roots.
- Creates "soft-link" like behavior that breaks tree navigation.

### Alternative 2: Remove `root` prefix entirely

Make the top-level workspace have no prefix, so workspaces appear as:
```
my-org:team:project  # instead of root:my-org:team:project
```

**Rejected because:**
- Breaking change affecting all existing deployments.
- `root` is referenced in many places as `CoreRootCluster`.
- Doesn't solve the sibling visibility problem.

### Alternative 3: Path translation/facade

Translate paths at the API layer so users see `a1b2c3:...` but internally
it's still `root:orgs:company-a:...`.

**Rejected because:**
- Security through obscurity - structure still exists.
- Complexity in maintaining translation layer.
- Doesn't provide true isolation.

## References

- GitHub Issue: https://github.com/kcp-dev/kcp/issues/3716
- Related KEP: [Decouple Logical Clusters from Hierarchy](decouple-logical-clusters-from-hierarchy.md)
- Related KEP: [Workspace Mounts](0002-proxy-workspace.md)
