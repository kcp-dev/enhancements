# Workspace Lifecycle Content Access Enhancement Proposal

## Summary

Replace the annotation-based workspace owner identity with a structured API field on
`LogicalCluster`, and introduce RBAC-gated impersonation for initializing and terminating
virtual workspace content proxies. Additionally, add workspace content access to the
terminating virtual workspace, which currently lacks it entirely.

## Motivation

The current workspace lifecycle has several gaps (see also
[kcp-dev/kcp#3647](https://github.com/kcp-dev/kcp/issues/3647)):

1. **Owner identity stored as annotation.** The workspace owner is serialized as JSON in the
   `experimental.tenancy.kcp.io/owner` annotation on the Workspace object
   (`staging/src/github.com/kcp-dev/sdk/apis/tenancy/v1alpha1/types_workspace.go:58-59`).
   The initializing virtual workspace content proxy reads this annotation from the LogicalCluster
   to impersonate the owner when forwarding requests
   (`pkg/virtual/initializingworkspaces/builder/build.go:241-254`). Annotations are untyped,
   unvalidated, and can be mutated by anyone with update access — making them unsuitable for
   security-critical identity data.

2. **Owner group information is wiped on Ready phase.** When a LogicalCluster transitions to
   `Ready`, the metadata reconciler
   (`pkg/reconciler/core/logicalcluster/logicalcluster_reconcile_metadata.go:99-117`) strips
   all fields except `Username` from the owner annotation. This means by the time a workspace
   reaches termination, the owner's groups, UID, and extra info are already lost. Even if the
   terminating virtual workspace attempted the same impersonation mechanism as the initializing
   VW, it would impersonate the owner without their groups — potentially breaking group-based
   RBAC evaluation. This is also not backwards compatible: workspaces created before any fix
   would already have lost the group data.

3. **Terminating workspaces have no content access.** The terminating virtual workspace
   (`pkg/virtual/terminatingworkspaces/builder/build.go`) only exposes LogicalCluster
   list/watch and status updates. Terminator controllers cannot access actual workspace
   content to clean up resources. The workspace content authorizer
   (`pkg/authorization/workspace_content_authorizer.go:112-114`) blocks all access for
   workspaces not in `Initializing` or `Ready` phase.

4. **No separate RBAC gate for workspace content access.** Currently, having the `initialize`
   verb on a workspacetype implicitly grants access to the workspace content proxy (for the
   initializing VW). There is no separate authorization step that distinguishes "can access
   LogicalCluster VW endpoints" from "can access workspace content." The content proxy
   impersonates the workspace owner — who has `cluster-admin` inside the workspace via the
   `workspace-admin` ClusterRoleBinding
   (`pkg/reconciler/tenancy/logicalcluster/logicalcluster_controller.go:236-250`) — but the
   controller itself may be a service account with limited permissions. Not every controller
   with VW access should automatically get this elevated content access.

### Goals

1. Move workspace owner identity from annotation to a structured, immutable field on
   `LogicalCluster`.
2. Introduce RBAC-gated impersonation on `WorkspaceType` so that controllers can be
   explicitly authorized to access workspace content with elevated privileges during
   initialization and termination. So not everybody with VW access automatically gets elevated access.
3. Add a workspace content proxy to the terminating virtual workspace, matching the
   capability that already exists for the initializing virtual workspace.

### Non-Goals

- General-purpose workspace content access outside of init/term phases. The `impersonate`
  verb on `workspacetypes` only applies to the virtual workspace content proxies, not to
  direct workspace access.
- Changing the workspace content authorizer phase gate. The authorizer continues to block
  direct access to non-`Initializing`/non-`Ready` workspaces. The virtual workspace proxies
  bypass this by hitting the shard directly with impersonation.
- Per-initializer or per-terminator impersonation identity selection. All controllers with
  the `impersonate` verb on a given WorkspaceType get the same elevated identity.

## Proposal

### Part 1: Structured Owner Identity on LogicalCluster

#### API Changes

Add a new type and field to `LogicalClusterSpec`
(`staging/src/github.com/kcp-dev/sdk/apis/core/v1alpha1/logicalcluster_types.go`):

```go
type LogicalClusterSpec struct {
    // ...existing fields (DirectlyDeletable, Owner, Initializers, Terminators)...

    // ownerUser is the identity of the user who created this workspace.
    // Set by the system at workspace creation time. Immutable after creation.
    // Used by virtual workspace content proxies to determine the default
    // impersonation identity during initialization and termination.
    //
    // +optional
    OwnerUser *LogicalClusterOwnerUser `json:"ownerUser,omitempty"`
}

// LogicalClusterOwnerUser holds the identity of the workspace creator.
type LogicalClusterOwnerUser struct {
    // username is the username of the user who created the workspace.
    //
    // +required
    // +kubebuilder:validation:Required
    // +kubebuilder:validation:MinLength=1
    Username string `json:"username"`

    // uid is the UID of the user.
    //
    // +optional
    UID string `json:"uid,omitempty"`

    // groups are the groups the user belongs to.
    //
    // +optional
    Groups []string `json:"groups,omitempty"`

    // extra contains additional user information.
    //
    // +optional
    Extra map[string]ExtraValue `json:"extra,omitempty"`
}

// ExtraValue masks the value so protobuf can generate.
type ExtraValue []string
```

#### Controller and Admission Changes

| Component | File | Change |
|---|---|---|
| Workspace admission | `pkg/admission/workspace/admission.go:95-108` | On workspace creation, set `ownerUser` on the Workspace (to be propagated to LogicalCluster) in addition to the existing annotation |
| LogicalCluster admission | `pkg/admission/logicalcluster/admission.go` | Validate that `spec.ownerUser` is immutable after creation |
| LogicalCluster controller | `pkg/reconciler/tenancy/logicalcluster/logicalcluster_controller.go:236-250` | Read owner from `spec.ownerUser` instead of annotation when creating `workspace-admin` CRB |
| Initializing VW content proxy | `pkg/virtual/initializingworkspaces/builder/build.go:241-254` | Read from `spec.ownerUser` instead of annotation; fall back to annotation for migration |

#### Migration

During transition, the content proxy reads `spec.ownerUser` first, falling back to the
`experimental.tenancy.kcp.io/owner` annotation. The annotation is deprecated and will be
removed in a future release.

### Part 2: RBAC-Gated Impersonation via WorkspaceType

#### Problem

An initializer or terminator controller is typically a service account with narrow permissions:
it can list/watch LogicalClusters via the virtual workspace and update their status. Currently,
having the `initialize` or `terminate` verb on a workspacetype grants access to the VW's
LogicalCluster endpoints, but also — implicitly and unconditionally — grants access to the
workspace content proxy (for initializing VW). There is no separate RBAC gate that controls
whether a controller is authorized to access workspace content.

Not every controller with VW access should automatically get workspace content access. The
`impersonate` verb provides an explicit, separate authorization gate for content access.

#### New RBAC Verb: `impersonate`

Introduce a new verb `impersonate` on the `workspacetypes` resource in the `tenancy.kcp.io`
API group. This verb controls whether a controller is allowed to access workspace content
through the virtual workspace content proxy. When granted, the proxy impersonates the workspace
owner (who has `cluster-admin` inside the workspace via the `workspace-admin` ClusterRoleBinding
created at `pkg/reconciler/tenancy/logicalcluster/logicalcluster_controller.go:236-250`).

The controller itself may be a service account with limited permissions. The `impersonate` verb
is what explicitly authorizes it to operate as the owner — and therefore as `cluster-admin` —
inside the workspace content.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: universal-initializer
rules:
  # Gate 1: Can access the initializing virtual workspace (LogicalCluster list/watch/status)
  - verbs: ["initialize"]
    resources: ["workspacetypes"]
    resourceNames: ["universal"]
    apiGroups: ["tenancy.kcp.io"]
  # Gate 2: Can access workspace content via proxy (impersonating the owner)
  - verbs: ["impersonate"]
    resources: ["workspacetypes"]
    resourceNames: ["universal"]
    apiGroups: ["tenancy.kcp.io"]
```

Similarly for terminators:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: universal-terminator
rules:
  - verbs: ["terminate"]
    resources: ["workspacetypes"]
    resourceNames: ["universal"]
    apiGroups: ["tenancy.kcp.io"]
  - verbs: ["impersonate"]
    resources: ["workspacetypes"]
    resourceNames: ["universal"]
    apiGroups: ["tenancy.kcp.io"]
```

A controller with only `initialize`/`terminate` (without `impersonate`) can list/watch
LogicalClusters and update their status, but **cannot** access workspace content.

#### Content Proxy Authorization Flow

When a controller accesses workspace content through the virtual workspace proxy:

```
1. Controller sends request to VW content endpoint
   /services/initializingworkspaces/<initializer>/clusters/<cluster>/<resource-path>

2. VW checks "initialize" verb on workspacetypes (existing)
   → Deny: 403 Forbidden
   → Allow: proceed

3. VW checks workspace phase and initializer presence (existing)
   → Phase != Initializing or initializer not in status.initializers: 403 Forbidden
   → OK: proceed

4. VW checks "impersonate" verb on workspacetypes (NEW)
   → Allow: proxy impersonates workspace owner from spec.ownerUser
   → Deny/NoOpinion: 403 Forbidden — content access not authorized
```

The same flow applies to the terminating virtual workspace, substituting `terminate` for
`initialize` and checking `deletionTimestamp` + `status.terminators`.

#### Impersonation Target

The proxy always impersonates the **workspace owner** from `spec.ownerUser` on the
LogicalCluster. The owner is bound to `cluster-admin` inside the workspace via the
`workspace-admin` ClusterRoleBinding
(`pkg/reconciler/tenancy/logicalcluster/logicalcluster_controller.go:236-250`), giving the
proxied request full access to workspace content.

The `impersonate` verb does not change who is impersonated — it gates whether the controller
is allowed to access content at all. This provides a clear separation:

- `initialize`/`terminate` → access to LogicalCluster VW endpoints
- `impersonate` → access to workspace content (as the owner)

#### Implementation Changes

| Component | File | Change |
|---|---|---|
| Initializing VW content proxy | `pkg/virtual/initializingworkspaces/builder/build.go:223-288` | Add delegated SAR check for `impersonate` verb before proxying; deny if not authorized |
| Terminating VW content proxy | New code (see Part 3) | Same `impersonate` check before proxying |
| ClusterRole replication | `pkg/reconciler/tenancy/replicateclusterrole/replicateclusterrole_controller.go:60` | Add `impersonate` to the set of verbs that trigger replication |

### Part 3: Terminating Virtual Workspace Content Proxy

#### Current State

The terminating virtual workspace (`pkg/virtual/terminatingworkspaces/builder/build.go:56-159`)
exposes two sub-workspaces:

| Name | Purpose |
|---|---|
| `wildcardLogicalClusters` | List/watch LogicalClusters with deletionTimestamp, filtered by terminator label |
| `logicalClusters` | Get/update individual LogicalCluster status (remove own terminator only) |

There is no `workspaceContent` sub-workspace. Terminators cannot access resources inside the
workspace to clean up before deletion.

#### Proposed Addition

Add a third sub-workspace `workspaceContent` to the terminating virtual workspace, mirroring
the pattern in the initializing virtual workspace
(`pkg/virtual/initializingworkspaces/builder/build.go:165-291`).

URL pattern:
```
/services/terminatingworkspaces/<terminator>/clusters/<cluster>/<resource-path>
```

#### Handler Logic

```
1. Parse terminator name from URL path (existing digestUrl function)

2. Reject if cluster is wildcard (content access requires specific cluster)

3. Reject if path is a logicalclusters.core.kcp.io request
   (handled by the existing logicalClusters sub-workspace)

4. Look up LogicalCluster from informer cache

5. Validate:
   - LogicalCluster has deletionTimestamp set
   - Terminator is present in status.terminators
   → Reject with 403 if either check fails

6. Check "impersonate" verb on workspacetypes for this terminator (Part 2)
   → Deny/NoOpinion: 403 Forbidden — content access not authorized
   → Allow: proceed

7. Read owner identity from spec.ownerUser on LogicalCluster

8. Reverse-proxy request to the workspace shard, impersonating the owner
```

#### Implementation Changes

| Component | File | Change |
|---|---|---|
| Terminating VW builder | `pkg/virtual/terminatingworkspaces/builder/build.go` | Add `workspaceContent` handler as third named VW; accept `wildcardKcpInformers` parameter |
| Terminating VW builder | `pkg/virtual/terminatingworkspaces/builder/build.go:61-66` | Add `workspaceContentName` constant |
| Terminating VW registration | Where `BuildVirtualWorkspace` is called | Pass `wildcardKcpInformers` (same as initializing VW does) |

The `workspaceContent` handler for the terminating VW is structurally identical to
`pkg/virtual/initializingworkspaces/builder/build.go:165-291`, with these differences:

- Checks `status.terminators` instead of `status.initializers`
- Checks `deletionTimestamp` is set instead of `phase == Initializing`
- Uses the `termination.TypeFrom` function instead of `initialization.TypeFrom`
- Requires `impersonate` verb to access content (same as initializing VW after this proposal)

## RBAC Summary

The following table summarizes the verbs on `workspacetypes` and what they gate:

| Verb | Resource | Purpose | Where Checked |
|---|---|---|---|
| `use` | `workspacetypes` | Can create workspaces of this type | Workspace admission |
| `initialize` | `workspacetypes` | Can access the initializing virtual workspace | Initializing VW authorizer |
| `terminate` | `workspacetypes` | Can access the terminating virtual workspace | Terminating VW authorizer |
| `impersonate` | `workspacetypes` | Can access workspace content via proxy (impersonating the owner) during init/term | VW content proxy (NEW) |

Example: a controller that initializes workspaces of type `tenant` and needs content access:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tenant-initializer
rules:
  - verbs: ["initialize", "impersonate"]
    resources: ["workspacetypes"]
    resourceNames: ["tenant"]
    apiGroups: ["tenancy.kcp.io"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tenant-initializer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tenant-initializer
subjects:
  - kind: ServiceAccount
    name: tenant-initializer-controller
    namespace: default
```

This ClusterRole and ClusterRoleBinding must exist in the workspace where the `tenant`
WorkspaceType is defined. The `replicateclusterrole` controller
(`pkg/reconciler/tenancy/replicateclusterrole/replicateclusterrole_controller.go:60`)
ensures ClusterRoles with `use`, `initialize`, `terminate`, or `impersonate` verbs on
`workspacetypes` are replicated across the hierarchy.

## Implementation Phases

```
Phase 1: Owner Identity (Part 1)
  - Add LogicalClusterOwnerUser type and ownerUser field to LogicalClusterSpec
  - Update workspace admission to populate ownerUser
  - Update logicalcluster admission to enforce immutability
  - Update logicalcluster controller to read ownerUser for workspace-admin CRB
  - Update initializing VW content proxy to read from spec.ownerUser
  - Deprecate experimental.tenancy.kcp.io/owner annotation

Phase 2: Impersonation RBAC (Part 2)
  - Add "impersonate" verb check to initializing VW authorizer and content proxy
  - Add "impersonate" to replicateclusterrole verb set
  - Add e2e tests for impersonation gating

Phase 3: Terminating VW Content Access (Part 3)
  - Add workspaceContent handler to terminating VW
  - Wire informers into terminating VW builder
  - Integrate impersonation RBAC from Phase 2
  - Add e2e tests for terminator content access
```

## Relationship to Existing Issues

This proposal addresses [kcp-dev/kcp#3647](https://github.com/kcp-dev/kcp/issues/3647)
("Allow accessing workspace content through terminating virtualworkspace").

That issue identified two core problems:
1. The owner annotation's group info is wiped when a workspace reaches Ready
   (`logicalcluster_reconcile_metadata.go:99-117`), making owner impersonation impossible
   for terminators.
2. The terminating VW has no content proxy at all.

The issue discussion considered several approaches:
- **Not wiping group data**: Would fix the immediate problem but is not backwards compatible
  for existing workspaces, and still couples content access to the owner identity.
- **Dynamically creating RBAC**: For controllers with `terminate`/`initialize` verbs, create
  ephemeral ClusterRoleBindings granting workspace content access. This adds complexity around
  lifecycle management of those RBAC objects.
- **God-like permissions for the VW**: Giving the terminating VW system-level access. Too
  broad and insecure.

This proposal takes a different approach: structured owner identity (solving the data-loss
problem) combined with RBAC-gated impersonation (decoupling content access from the owner
entirely). The `impersonate` verb on workspacetypes provides explicit, auditable authorization
for elevated access without ephemeral RBAC objects or annotation hacks.

### Backwards Compatibility

For workspaces created before this enhancement:
- **Owner identity**: The `spec.ownerUser` field will not be set. The content proxy falls back
  to the annotation. If the annotation has already been stripped to username-only (workspace
  reached Ready), the owner can still be impersonated by username — the `workspace-admin`
  ClusterRoleBinding (`pkg/reconciler/tenancy/logicalcluster/logicalcluster_controller.go:236-250`)
  binds by username, not by group, so `cluster-admin` access still works.
- **Impersonate verb**: Controllers with the `impersonate` verb bypass the owner identity
  entirely, using the `kcp-admin` system identity. This works regardless of whether
  `spec.ownerUser` or the annotation exists.
- **Group-wiping removal**: As part of Phase 1, the group-wiping logic in
  `logicalcluster_reconcile_metadata.go:99-117` should be removed. With owner identity moving
  to an immutable spec field, there is no longer a reason to strip data from the annotation.

## Breaking Changes

This enhancement introduces a **breaking change** in the authorization model for virtual
workspace content access.

### Before

Any controller with the `initialize` verb on a `workspacetypes` resource could access
workspace content through the initializing virtual workspace content proxy. The `initialize`
verb served as a single gate for both LogicalCluster VW endpoints (list/watch/status) and
workspace content (creating namespaces, configmaps, etc. inside the workspace). There was no
separate authorization step for content access — it was implicitly granted.

### After

Content access through the virtual workspace content proxy now requires the **`impersonate`**
verb on the `workspacetypes` resource **in addition to** `initialize` or `terminate`.

- `initialize` or `terminate` alone → access to LogicalCluster VW endpoints only
- `initialize` + `impersonate` → access to LogicalCluster VW endpoints **and** workspace content
- `terminate` + `impersonate` → access to LogicalCluster VW endpoints **and** workspace content

### Impact

**Existing initializer controllers that access workspace content will break** unless their
ClusterRoles are updated to include the `impersonate` verb. Controllers that only list/watch
LogicalClusters and update their status (i.e., do not access workspace content) are unaffected.

**Required migration**: For any ClusterRole that grants `initialize` or `terminate` on
`workspacetypes` and whose controller accesses workspace content through the VW proxy, add
`impersonate` to the verbs:

```yaml
# Before
rules:
  - verbs: ["initialize"]
    resources: ["workspacetypes"]
    resourceNames: ["my-type"]
    apiGroups: ["tenancy.kcp.io"]

# After
rules:
  - verbs: ["initialize", "impersonate"]
    resources: ["workspacetypes"]
    resourceNames: ["my-type"]
    apiGroups: ["tenancy.kcp.io"]
```

The same applies to `terminate`:

```yaml
# Before
rules:
  - verbs: ["terminate"]
    resources: ["workspacetypes"]
    resourceNames: ["my-type"]
    apiGroups: ["tenancy.kcp.io"]

# After
rules:
  - verbs: ["terminate", "impersonate"]
    resources: ["workspacetypes"]
    resourceNames: ["my-type"]
    apiGroups: ["tenancy.kcp.io"]
```

## Open Questions

1. **Should `impersonate` without `initialize`/`terminate` be meaningful?**
   Currently, `impersonate` only applies to controllers that already have VW access via
   `initialize` or `terminate`. The content proxy requires both: VW access (`initialize`/
   `terminate`) AND content access (`impersonate`). Should `impersonate` alone ever grant
   anything?

2. **Annotation removal timeline.**
   How long should the `experimental.tenancy.kcp.io/owner` annotation be supported as a
   fallback before removal?

3. **Should the group-wiping logic be removed entirely or made conditional?**
   The current code strips group/UID/extra from the owner annotation on Ready. With
   `spec.ownerUser` as the source of truth, the annotation becomes redundant. Should
   we stop wiping immediately, or keep the behavior until the annotation is fully deprecated?

4. **Should the existing initializing VW content proxy require `impersonate`?**
   Today, any controller with the `initialize` verb can access workspace content implicitly.
   Adding the `impersonate` requirement is a breaking change for existing initializer
   controllers. Should this be enforced immediately or introduced with a deprecation period
   where the old behavior (implicit content access with `initialize` only) is preserved?
