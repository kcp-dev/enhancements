# Workspace Lifecycle Content Access Enhancement Proposal

## Summary

Replace the annotation-based workspace owner identity with a structured API field on
`LogicalCluster`, introduce fine-grained RBAC-based workspace content access for initializing
and terminating virtual workspace content proxies, add a `Terminating` phase for visibility,
and add workspace content access to the terminating virtual workspace.

## Motivation

The current workspace lifecycle has several gaps (see also
[kcp-dev/kcp#3647](https://github.com/kcp-dev/kcp/issues/3647)):

1. **Owner identity stored as annotation.** The workspace owner is serialized as JSON in the
   `experimental.tenancy.kcp.io/owner` annotation on the Workspace object. Annotations are
   untyped, unvalidated, and can be mutated by anyone with update access — making them
   unsuitable for security-critical identity data.

2. **Owner group information is wiped on Ready phase.** When a LogicalCluster transitions to
   `Ready`, the metadata reconciler strips all fields except `Username` from the owner
   annotation. By the time a workspace reaches termination, the owner's groups, UID, and extra
   info are already lost.

3. **Terminating workspaces have no content access.** The terminating virtual workspace only
   exposes LogicalCluster list/watch and status updates. Terminator controllers cannot access
   actual workspace content to clean up resources. The workspace content authorizer blocks all
   access for workspaces not in `Initializing` or `Ready` phase.

4. **No fine-grained permissions for workspace content access.** The initializing VW content
   proxy impersonates the workspace owner — who has `cluster-admin` inside the workspace.
   Having `initialize` on a workspacetype implicitly grants full cluster-admin content access.
   There is no way to scope an initializer's permissions (e.g., read-only configmaps). Not
   every controller needs full admin access to workspace content.

5. **Misleading audit logs.** The content proxy impersonates the workspace owner identity.
   Audit logs cannot distinguish whether a request came from the actual owner or a controller
   impersonating them. With multiple initializers/terminators, controllers are
   indistinguishable from each other and from the owner. Clear attribution is a first-class
   goal of this proposal.

6. **No `Terminating` phase.** Workspaces that are being terminated show as `Ready` in
   `kubectl get workspaces`. There is no way to tell which workspaces are stuck in termination
   without inspecting deletion timestamps and terminator lists.

All initializing/terminating VW requests continue to be routed through the FrontProxy (not
directly to shards), as today. This is unchanged by this proposal — the VW machinery is a
replicated, front-proxy-served concern by design.

### Goals

1. Move workspace owner identity from annotation to a structured, immutable field on
   `LogicalCluster`.
2. Introduce fine-grained, declarative RBAC for workspace content access during initialization
   and termination, defined on `WorkspaceType`. Granularity matches standard Kubernetes RBAC
   (apiGroups / resources / verbs / resourceNames). Controllers access content with their own
   identity, scoped to explicitly declared permissions.
3. Clear audit attribution: initializer/terminator requests carry the controller's own
   identity, not the workspace owner's, so audit logs unambiguously identify which controller
   acted.
4. Add a workspace content proxy to the terminating virtual workspace, matching the capability
   that already exists for the initializing virtual workspace.
5. Add a `Terminating` phase for workspace visibility.
6. Preserve backwards compatibility: WorkspaceTypes without explicit permissions fall back to
   owner impersonation (current behavior).

### Non-Goals

- General-purpose workspace content access outside of init/term phases.
- Changing direct workspace access patterns. The workspace content authorizer continues to
  handle direct access; the VW proxies handle lifecycle content access.
- Per-controller custom impersonation identity selection.
- Per-workspace (instance-level) overrides of initializer/terminator permissions. Permissions
  are declared once on the WorkspaceType and apply uniformly to all workspaces of that type.

## Proposal

### Part 1: Structured Owner Identity on LogicalCluster

#### API Changes

Add a new type and field to `LogicalClusterSpec`:

```go
type LogicalClusterSpec struct {
    // ...existing fields (DirectlyDeletable, Owner, Initializers, Terminators)...

    // ownerUser is the identity of the user who created this workspace.
    // Set by the system at workspace creation time. Immutable after creation.
    // Used as impersonation identity when WorkspaceType does not define
    // explicit initializerPermissions/terminatorPermissions, and for
    // creating the workspace-admin ClusterRoleBinding.
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
| Workspace admission | `pkg/admission/workspace/admission.go` | On workspace creation, set `ownerUser` on the Workspace (to be propagated to LogicalCluster); stop setting the `experimental.tenancy.kcp.io/owner` annotation |
| LogicalCluster admission | `pkg/admission/logicalcluster/admission.go` | Validate that `spec.ownerUser` is immutable after creation |
| LogicalCluster controller | `pkg/reconciler/tenancy/logicalcluster/logicalcluster_controller.go` | Read owner from `spec.ownerUser` when creating `workspace-admin` CRB |
| Metadata reconciler | `pkg/reconciler/core/logicalcluster/logicalcluster_reconcile_metadata.go` | Remove group-wiping logic and annotation handling entirely |
| Initializing VW content proxy | `pkg/virtual/initializingworkspaces/builder/build.go` | Read from `spec.ownerUser` only; remove annotation fallback |

The `experimental.tenancy.kcp.io/owner` annotation is removed entirely in this change.
There is no fallback — `spec.ownerUser` is the sole source of truth. Existing workspaces
without `spec.ownerUser` set will not have owner identity available for impersonation;
they must be recreated.

### Part 2: Declarative RBAC for Workspace Content Access

#### Problem

The current content proxy impersonates the workspace owner, giving controllers full
cluster-admin access. This has several problems:

- **No fine-grained permissions.** An initializer that only needs to create ConfigMaps gets
  full admin access to all resources in the workspace.
- **Misleading audit logs.** All controllers appear as the workspace owner. Multiple
  initializers/terminators are indistinguishable from each other and from the real owner.
- **Implicit access.** Any controller with `initialize` automatically gets content access.

#### Solution: Permissions on WorkspaceType

Add optional permission fields to `WorkspaceTypeSpec` that declare what initializer and
terminator controllers can do inside workspaces of this type:

```go
type WorkspaceTypeSpec struct {
    // ...existing fields...

    // initializerPermissions defines the RBAC rules to grant to initializer
    // controllers inside workspaces of this type during initialization.
    // If empty, the content proxy falls back to owner impersonation
    // (full cluster-admin, backwards compatible).
    //
    // +optional
    InitializerPermissions []rbacv1.PolicyRule `json:"initializerPermissions,omitempty"`

    // terminatorPermissions defines the RBAC rules to grant to terminator
    // controllers inside workspaces of this type during termination.
    // If empty, the content proxy falls back to owner impersonation
    // (full cluster-admin, backwards compatible).
    //
    // +optional
    TerminatorPermissions []rbacv1.PolicyRule `json:"terminatorPermissions,omitempty"`
}
```

Example WorkspaceType with fine-grained permissions:

```yaml
apiVersion: tenancy.kcp.io/v1alpha1
kind: WorkspaceType
metadata:
  name: tenant
spec:
  initializer: true
  terminator: true
  initializerPermissions:
    - apiGroups: [""]
      resources: ["configmaps", "secrets", "namespaces"]
      verbs: ["get", "list", "create", "update", "delete"]
    - apiGroups: ["rbac.authorization.k8s.io"]
      resources: ["clusterroles", "clusterrolebindings"]
      verbs: ["get", "list", "create"]
  terminatorPermissions:
    - apiGroups: [""]
      resources: ["*"]
      verbs: ["get", "list", "delete"]
```

Example WorkspaceType without explicit permissions (backwards compatible):

```yaml
apiVersion: tenancy.kcp.io/v1alpha1
kind: WorkspaceType
metadata:
  name: legacy
spec:
  initializer: true
  # No initializerPermissions → falls back to owner impersonation
```

#### Two Modes of Content Access

The VW content proxy operates in two modes based on whether the WorkspaceType defines
explicit permissions:

**Mode 1: Explicit permissions (new, preferred)**

When `initializerPermissions` or `terminatorPermissions` is set on the WorkspaceType:

1. The VW proxy evaluates the incoming request **in-process** against the WorkspaceType's
   `initializerPermissions` / `terminatorPermissions` rules (verb / apiGroup / resource /
   resourceName match) using the standard RBAC evaluation logic. This happens after the
   existing `initialize`/`terminate` verb check and phase/presence validation.
2. If the request is denied, the VW proxy returns 403 without ever forwarding to the shard.
3. If allowed, the VW proxy forwards to the shard with the **controller's own identity**
   plus a synthetic group (see "Synthetic Groups" below).
4. On the shard side, the workspace content authorizer recognizes the synthetic group as a
   "pre-authorized by VW" signal and allows the request. It does **not** re-evaluate the
   WorkspaceType rules — the VW has already done that. This is a trivial trust check, not a
   full authorizer.
5. Audit logs show the controller's actual identity — clear attribution.

This keeps the expensive work (rule evaluation) on the lifecycle path only: the VW proxy
is the single place that knows about `initializerPermissions` / `terminatorPermissions`.
The normal workspace authorization chain is **not** modified to evaluate these rules — it
only needs to recognize and trust the synthetic group when set, and reject it when not.

This is deliberately "magic/ephemeral" rather than "visible" RBAC: the rules live on the
WorkspaceType (the source of truth) and are evaluated by the VW proxy on each request.
This avoids:

- Running lifecycle permission logic for every workspace request — only VW-bound requests
  pay the cost.
- Lifecycle permissions accidentally applying outside init/term phases — the VW proxy is
  the only entry point that knows about these rules; direct shard requests cannot benefit.
- Race conditions at workspace creation ("is the CRB there yet when the controller hits the
  API?") — nothing needs to be created in the workspace before the controller can act.
- Users accidentally or deliberately deleting the CRB and breaking lifecycle access.
- Drift between WorkspaceType rules and materialized objects.
- Finalizers/cleanup machinery for CRBs that get left behind after initialization.

**Mode 2: Owner impersonation (fallback, backwards compatible)**

When `initializerPermissions` or `terminatorPermissions` is empty/nil:

1. The VW proxy impersonates the workspace owner from `spec.ownerUser`.
2. The request reaches the workspace as the owner, who has cluster-admin via the
   `workspace-admin` ClusterRoleBinding.
3. This is the current behavior — no changes needed for existing controllers.

#### Synthetic Groups

For each initializer/terminator, the VW proxy injects a well-known group:

- Initializers: `system:kcp:initializer:<initializer-name>`
- Terminators: `system:kcp:terminator:<terminator-name>`

The `<initializer-name>` / `<terminator-name>` is the fully qualified identifier produced by
`initialization.InitializerForType` / `termination.TerminatorForType`, which encodes both the
workspace path and the type name (e.g., `root:org:tenant`). This is already globally unique —
two WorkspaceTypes with the same name in different workspaces produce different identifiers.
No clash is possible.

These groups are only added by the VW proxy — they cannot be self-asserted by clients.
This is enforced by the front-proxy's existing group filtering mechanism: the
`--authentication-drop-groups` default list is extended to include `system:kcp:initializer:*`
and `system:kcp:terminator:*`. This strips these groups from **all** client-asserted requests
entering the front-proxy, before any routing occurs. The VW proxy (which runs inside the
front-proxy process) then adds the synthetic groups **after** auth filtering when constructing
the forwarded request to the shard. This is the same pattern used for
`system:kcp:logical-cluster-admin` today — dropped at the front-proxy gate, injected only
by trusted internal code paths.

#### Ephemeral (in-memory) RBAC Evaluation

When a WorkspaceType has explicit `initializerPermissions` / `terminatorPermissions`, kcp
does **not** create any ClusterRole or ClusterRoleBinding objects inside the workspace.
Instead, the VW proxy evaluates the request in-process against the WorkspaceType spec
before forwarding to the shard.

Conceptually the evaluation is equivalent to what RBAC would do if the following objects
existed — but they do **not** exist as API objects, they are evaluated in memory from the
WorkspaceType spec at request time:

```yaml
# Virtual, in-memory only — NOT a real object in the workspace
ClusterRole: system:kcp:initializer:<initializer-name>
  rules: <from WorkspaceType.spec.initializerPermissions>
ClusterRoleBinding: system:kcp:initializer:<initializer-name>
  roleRef: system:kcp:initializer:<initializer-name>
  subjects:
    - kind: Group
      name: system:kcp:initializer:<initializer-name>
```

The VW proxy's evaluator:

1. Resolves the WorkspaceType for the current workspace (via `LogicalCluster.spec.type` /
   cached WorkspaceType informer).
2. Evaluates the request's verb / apiGroup / resource / resourceName against the matching
   `initializerPermissions` or `terminatorPermissions` rules using standard RBAC matching
   semantics.
3. Allows → forward to shard with controller identity + synthetic group.
   Denies → return 403 immediately.

Because rules live on the WorkspaceType and are evaluated on the fly, permission changes on
the WorkspaceType take effect immediately for all workspaces of that type. There is no
snapshot, no drift, no materialized state to reconcile. See the "Permission updates" note
in Open Questions for the policy trade-off.

#### Extending Workspace Types

When `gamma` extends `alpha` and `beta`, each initializer/terminator is independent. A
workspace of type `gamma` is evaluated against three separate sets of rules — one per
initializer's synthetic group, using that initializer's permissions from its own
WorkspaceType. No merging. The authorizer looks up permissions from the WorkspaceType that
**owns** the synthetic group, not from the workspace's own type.

#### Content Proxy Authorization Flow

```
1. Controller sends request to VW content endpoint
   /services/initializingworkspaces/<initializer>/clusters/<cluster>/<resource-path>

2. VW checks "initialize" verb on workspacetypes (existing)
   → Deny: 403 Forbidden
   → Allow: proceed

3. VW checks workspace phase and initializer presence (existing)
   → Phase != Initializing or initializer not in status.initializers: 403 Forbidden
   → OK: proceed

4. VW checks WorkspaceType for explicit permissions
   → initializerPermissions set:
       a. Evaluate request in-process against initializerPermissions rules
          - Denied: return 403 immediately, do not forward
          - Allowed: forward to shard with controller identity + synthetic group
   → initializerPermissions empty: impersonate workspace owner (fallback)

5. Shard receives forwarded request
   → Workspace content authorizer sees synthetic group system:kcp:initializer:<name>:
     allow (trust: VW has already authorized; only VW can inject this group)
   → No WorkspaceType rule evaluation on the shard
```

The same flow applies to the terminating virtual workspace, substituting `terminate` for
`initialize` and checking `deletionTimestamp` + `status.terminators`.

#### Workspace Content Authorizer Changes

The workspace content authorizer (`pkg/authorization/workspace_content_authorizer.go`)
currently allows access only for `Initializing` and `Ready` phases. Changes needed:

- Allow access for the `Terminating` phase (see Part 4).
- Recognize synthetic groups (`system:kcp:initializer:*`, `system:kcp:terminator:*`) as a
  "pre-authorized by VW" marker and allow the request. This is a trivial trust check — the
  shard does not evaluate WorkspaceType rules, it delegates that entirely to the VW proxy.
  Synthetic groups cannot be self-asserted by clients because the front-proxy's
  `--authentication-drop-groups` strips them from all incoming requests before routing (see
  "Synthetic Groups" above).

No new authorizer is added to the workspace authorization chain. The heavy lifting of
lifecycle permission evaluation is confined to the VW proxy, which means:

- Normal (non-lifecycle) workspace requests are unaffected — no extra work per request.
- Lifecycle permissions only apply to requests arriving via the VW proxy, which only
  accepts them during `Initializing` / `Terminating` phases with a matching entry in
  `status.initializers` / `status.terminators`. Access cannot leak outside those phases.

#### Implementation Changes

| Component | File | Change |
|---|---|---|
| WorkspaceType API | SDK `apis/tenancy/v1alpha1/types_workspacetype.go` | Add `InitializerPermissions` and `TerminatorPermissions` fields |
| Initializing VW content proxy | `pkg/virtual/initializingworkspaces/builder/build.go` | Two modes: in-process RBAC evaluation against `initializerPermissions` then forward with synthetic group (explicit perms), or owner impersonation (fallback) |
| Terminating VW content proxy | `pkg/virtual/terminatingworkspaces/builder/build.go` | Same two-mode logic for `terminatorPermissions` |
| Workspace content authorizer | `pkg/authorization/workspace_content_authorizer.go` | Allow `Terminating` phase; trust synthetic groups as pre-authorized |
| Front-proxy group filtering | `pkg/proxy/options/authentication.go` | Add `system:kcp:initializer:*` and `system:kcp:terminator:*` to `--authentication-drop-groups` defaults |

### Part 3: Terminating Virtual Workspace Content Proxy

#### Current State

The terminating virtual workspace exposes two sub-workspaces:

| Name | Purpose |
|---|---|
| `wildcardLogicalClusters` | List/watch LogicalClusters with deletionTimestamp, filtered by terminator label |
| `logicalClusters` | Get/update individual LogicalCluster status (remove own terminator only) |

There is no `workspaceContent` sub-workspace.

#### Proposed Addition

Add a third sub-workspace `workspaceContent` to the terminating virtual workspace, mirroring
the pattern in the initializing virtual workspace.

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
6. Check WorkspaceType for explicit terminatorPermissions
   → Set: inject synthetic group, forward with controller identity
   → Empty: impersonate workspace owner (fallback)
7. Reverse-proxy request to the workspace shard
```

The handler is structurally identical to the initializing VW content proxy, with:

- Checks `status.terminators` instead of `status.initializers`
- Checks `deletionTimestamp` is set instead of `phase == Initializing`
- Uses `termination.TypeFrom` instead of `initialization.TypeFrom`
- Reads `terminatorPermissions` instead of `initializerPermissions`

### Part 4: Terminating Phase

#### Current State

Workspaces have these phases: `Scheduling`, `Initializing`, `Ready`. A workspace that is
being terminated still shows `Ready` — there is no way to distinguish active workspaces
from stuck-terminating ones in `kubectl get workspaces`.

#### Change

Add `LogicalClusterPhaseTerminating` to the phase enum. The metadata reconciler sets this
phase when a workspace has a `deletionTimestamp`.

The workspace content authorizer is updated to allow content access for the `Terminating`
phase (in addition to `Initializing` and `Ready`).

## RBAC Summary

The following table summarizes the verbs on `workspacetypes` and what they gate:

| Verb | Resource | Purpose | Where Checked |
|---|---|---|---|
| `use` | `workspacetypes` | Can create workspaces of this type | Workspace admission |
| `initialize` | `workspacetypes` | Can access the initializing virtual workspace | Initializing VW authorizer |
| `terminate` | `workspacetypes` | Can access the terminating virtual workspace | Terminating VW authorizer |

Content access scope is controlled declaratively on the WorkspaceType, not via additional
RBAC verbs.

Example: a controller that initializes workspaces of type `tenant`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: tenant-initializer
rules:
  - verbs: ["initialize"]
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

The `initialize` verb grants VW access. Content access scope is determined by the
WorkspaceType's `initializerPermissions`. If empty, falls back to owner impersonation
(cluster-admin).

## Implementation Phases

```
Phase 1: Owner Identity and Annotation Removal
  - Add LogicalClusterOwnerUser type and ownerUser field to LogicalClusterSpec
  - Update workspace admission to populate ownerUser and stop setting annotation
  - Update logicalcluster admission to enforce immutability
  - Update logicalcluster controller to read ownerUser for workspace-admin CRB
  - Update initializing VW content proxy to read from spec.ownerUser only
  - Remove experimental.tenancy.kcp.io/owner annotation entirely
  - Remove group-wiping logic from metadata reconcilers

Phase 2: Terminating Phase and Content Proxy
  - Add LogicalClusterPhaseTerminating
  - Update metadata reconciler to set Terminating phase on deletion
  - Update workspace content authorizer to allow Terminating phase
  - Add workspaceContent handler to terminating VW
  - Wire informers into terminating VW builder
  - Content proxy uses owner impersonation (same as initializing VW)
  - Add e2e tests for terminator content access

Phase 3: Declarative RBAC on WorkspaceType (ephemeral)
  - Add initializerPermissions/terminatorPermissions to WorkspaceTypeSpec
  - Implement in-process RBAC evaluation in the initializing/terminating VW content
    proxies (evaluate request against WorkspaceType permissions before forwarding)
  - Implement synthetic group injection when forwarding
  - Update workspace content authorizer to trust synthetic groups as pre-authorized
  - Add synthetic groups to front-proxy `--authentication-drop-groups` defaults
    to prevent client self-assertion
  - Fallback to owner impersonation when permissions are empty
  - Add e2e tests for fine-grained permissions
```

## Breaking Changes

**Phase 1: Annotation removal.** The `experimental.tenancy.kcp.io/owner` annotation is
removed entirely. Code that reads owner identity from the annotation must switch to
`spec.ownerUser`. Existing workspaces without `spec.ownerUser` set will not have owner
identity available — they must be recreated.

**Phase 2: Terminating phase.** Workspaces that are being deleted will transition from
`Ready` to `Terminating`. Code that checks for `phase == Ready` to determine if a workspace
is usable may need to account for this new phase. The workspace content authorizer is updated
to allow both phases.

Phase 3 is **not a breaking change** for existing controllers. WorkspaceTypes without
explicit `initializerPermissions`/`terminatorPermissions` fall back to owner impersonation,
preserving the current behavior. Controllers only need to be updated when WorkspaceType
authors opt into fine-grained permissions.

## Backwards Compatibility

- **Owner identity**: The `experimental.tenancy.kcp.io/owner` annotation is removed. Existing
  workspaces without `spec.ownerUser` will not have owner identity available for the
  impersonation fallback. This is a breaking change — such workspaces must be recreated.
  The `workspace-admin` CRB is unaffected (it binds by username from the spec field).
- **WorkspaceType permissions**: Empty permissions = owner impersonation = current behavior.
  No changes needed for existing WorkspaceTypes or controllers.
- **Terminating phase**: New phase value. Code that only checks `Ready` continues to work
  for active workspaces. Code that needs to distinguish active from terminating can check
  for the new phase.

## Open Questions & Follow-ups

1. **Permission validation.** Should kcp validate that `initializerPermissions` /
   `terminatorPermissions` only reference API groups and resources that are available in the
   workspace? Or allow any rules and let RBAC evaluation handle mismatches?

2. **Permission updates — semantics of ephemeral evaluation.** Because rules are evaluated
   live from the WorkspaceType, edits to the WorkspaceType take effect **immediately** for
   all existing workspaces of that type. This differs from the `defaultAPIBindingLifecycle`
   `InitializeOnly` vs `Maintain` split, which only matters for materialized state. With
   ephemeral RBAC, there is no materialized state to snapshot — the WorkspaceType **is** the
   state. This is intentional: one source of truth, no drift, no reconciler. Operators must
   be aware that tightening permissions on a WorkspaceType is an immediate change for every
   workspace of that type.
