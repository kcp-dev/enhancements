# Workspace Mounts Enhancement Proposal

## Summary

To extend the spectrum of use cases for `kcp`, we need to support workspaces that
are not backed by a `kcp` cluster itself but are backed by a mount that can be used
to proxy requests directly to remote clusters or other workspaces. This enables
use cases where external and internal integrations can be added into the same workspace structure.

## Motivation

This would enable use cases where `kcp` is used as a proxy to remote clusters, but
some of the infrastructure is still managed by `kcp`. For example, `kcp` could be
responsible for ArgoCD, Crossplane, or other infrastructure that is more CRD-based.

At this point, communication between `kcp` and remote clusters is done via workspace abstractions.

For example:
```
$ kubectl ws tree
root
  ├── clusters (workspace)
  │   ├── cluster-mount (normal workspace, with mounted cluster ontop of it)
  │   ├── cluster-1 (workspace)
  ├── infra (workspace)
  │   ├── argocd (workspace)
  │   ├── crossplane (workspace)
  ...
```

Where `cluster-mount` is a workspace backed by a mount that can be used to proxy
requests to remote clusters. This provides a unified way of interacting with
remote clusters while still having the ability to manage some of the infrastructure
via `kcp`.

### Goals

1. Mounting remote clusters as workspaces.
2. Softlink - mounting other workspaces as sub-workspaces, where changing the
   sub-workspace would change the parent workspace as well.
3. Workspace status propagation - when mounting a workspace, there should be a
   way to propagate the status to the workspace itself, preventing false positives.

### Non-Goals

* Proxy/Mount functionality is not meant to replace `kcp` clusters. Additionally,
  the implementation of the mount functionality itself is out of the scope of this
  proposal. We should enable the mount functionality to be implemented by third
  parties using `kcp` as a platform.

* Mount resources. This proposal is about enabling the mount functionality for
  all workspaces, not about mounting individual resources inside a workspace.

* AuthN/AuthZ is out of scope of this proposal. We assume that the mount functionality
  implements just forwarding and redirections. Authentication and authorization are
  the implementer's responsibility, either by federating identities or using other
  means to authenticate and authorize requests.

## Proposal

Introduce an experimental `Mount` API to enable two use cases:

1. Mounting remote clusters as workspaces.
2. Softlink - mounting other workspaces as sub-workspaces, where changing the
   sub-workspace would change the parent workspace as well.

Initially, we would put Mount into the annotation of the workspace, but in the
future, we will promote it to the spec of the workspace.

### Mount API

This is a proposal for a mount API. This contains two parts:

1. Experimental mount API used in the annotation of the workspace.
2. Proposal for how this could look when promoted to the workspace structure.


#### Experimental Mount API

As an experimental API, we would place the mount into the annotation of the workspace.

```go
// Mount is a workspace mount that can be used to mount a workspace into another workspace or resource.
// Mounting itself is done at the front proxy level.
type Mount struct {
	// MountSpec is the spec of the mount.
	MountSpec MountSpec `json:"spec,omitempty"`
	// MountStatus is the status of the mount.
	MountStatus MountStatus `json:"status,omitempty"`
}

type MountSpec struct {
	// Reference is an ObjectReference to the object that is mounted.
	Reference *corev1.ObjectReference `json:"ref,omitempty"`
	// Type is the type of the mount (URL, Redirect, Workspace).
	// +kubebuilder:default=Cluster
	Type MountType `json:"type,omitempty"`
}

// MountStatus is the status of a mount. It is used to indicate the status of a mount,
// potentially managed outside of the core API.
type MountStatus struct {
	// Phase of the mount (Initializing, Connecting, Ready, Unknown).
	//
	// +kubebuilder:default=Initializing
	Phase MountPhaseType `json:"phase,omitempty"`
	// Conditions is a list of conditions and their status.
	// Current processing state of the Mount.
	// +optional
	Conditions conditionsv1alpha1.Conditions `json:"conditions,omitempty"`

	// URL is the URL of the mount. Mount is considered mounted when the URL is set.
	// +optional
	URL string `json:"url,omitempty"`
}
```

#### Promoted Workspace API

When the mount is promoted to the workspace spec, it could look like the example
below. Mount would be just yet another condition on the workspace. The most important
change is that the mount would be part of the workspace spec, and if the mount
implementation becomes unhealthy, it would be reflected in the workspace conditions.
This would drive the `Phase` change of the workspace. More on this in the next section.

```go
// WorkspaceSpec defines the desired state of Workspace
type WorkspaceSpec struct {
	...
	// Mount is a workspace mount that can be used to mount a workspace into another workspace or resource.
	Mount *Mount `json:"mount,omitempty"`
}

// Mount is a workspace mount that can be used to mount a workspace into another workspace or resource.
type Mount struct {
	// Reference is an ObjectReference to the object that is mounted.
	Reference *corev1.ObjectReference `json:"ref,omitempty"`
	// Type is the type of the mount (URL, Redirect, Workspace).
	// +kubebuilder:default=Cluster
	Type MountType `json:"type,omitempty"`
}

// WorkspaceStatus defines the observed state of Workspace
type WorkspaceStatus struct {
	// Phase of the workspace (Scheduling, Initializing, Ready).
	//
	// +kubebuilder:default=Scheduling
	Phase corev1alpha1.LogicalClusterPhaseType `json:"phase,omitempty"`

	// Current processing state of the Workspace.
	// +optional
	Conditions conditionsv1alpha1.Conditions `json:"conditions,omitempty"`

	// Initializers must be cleared by a controller before the workspace is ready
	// and can be used.
	//
	// +optional
	Initializers []corev1alpha1.LogicalClusterInitializer `json:"initializers,omitempty"`

	// MountURL is the URL of the mount. Mount is considered mounted when the URL is set.
	// +optional
	MountURL string `json:"mountURL,omitempty"`
}
```

#### Type explanations

Where `MountType` is an enum that would allow specifying the type of the mount:

- `Proxy` - the mount is backed by a URL and should be used as a proxy to a remote cluster.
- `Redirect` - the mount is backed by another workspace and should be used as a redirect to another workspace.
- `LogicalCluster` - the mount is backed by another workspace and should be used as a proxy to another workspace/logical cluster.

`Proxy` most simple proxy implementation is URL where we would just proxy all requests.
A more advanced implementation could include `VirtualWorkspace` or an external
service. But from a `kcp` perspective, it's a reverse proxy to a configured URL.

`Redirect` and `LogicalCluster` have two main behaviors. These include softlinking
of the (external or internal).

1. Case 1: Local redirect:

As a user, I want to redirect workspace `:users:john` to `:my-org:dev:john` so
that I can have a redirect inside the same instance of `kcp`. In this case, the
front proxy returns a `301 Redirect` request with a local (hostless) path to the
new workspace. The CLI should be able to follow the redirect and update the local
kubeconfig to point to the new workspace.

1. Case 2: External redirect:

As a user, I want to redirect workspace `:users:john` to `https://kcp.mycompany.com/workspaces/my-org/dev/john`
so that I can have a redirect to another `kcp` instance. In this case, the front proxy
returns a `301 Redirect` request with a full URL to the new workspace. The CLI
should be able to follow the redirect and update the local kubeconfig. This
would allow nesting different `kcp` instances.

`Reference` is a reference to the object that is mounted. It can be a reference
to an external object used to back the mount.

Object reference would be read by the `kcp` core mount controller. It will retrieve
the proxy URL from the referenced object's status and copy it into the mount status.
This way, third-party components never need to interact with the `kcp` core or workspace
directly in write mode. Moreover, the front-proxy interpreting the mount status URL
does not have to read these mount implementation objects directly either.

```yaml
apiVersion: proxy.contrib.kcp.io/v1alpha1
kind: WorkspaceProxy
metadata:
   name: cluster-proxy
spec:
   type: passthrough
```

### Workspace & Mount Status Propagation

When mounting a workspace, there should be a way to propagate the status to the
workspace itself, ensuring that the workspace is not falsely marked as ready when
the mount is not ready.

When a workspace mount is implemented by external objects:
```go
	// Reference is an ObjectReference to the object that is mounted.
	Reference *corev1.ObjectReference `json:"ref,omitempty"`
```

We need a way to propagate the status from the mount object to the workspace object.

The suggestion is to create a `kcp` core controller which would propagate any
conditions from the mount object to the workspace object which starts with the `Workspace*` prefix.

If an object, implementing object raises conditions:

```yaml
    conditions:
    - lastTransitionTime: "2024-04-28T10:43:29Z"
      status: "False"
      type: ConnectorReady
	- lastTransitionTime: "2024-04-28T10:43:29Z"
	  status: "False"
	  type: WorkspaceMountReady
```
it would be propagated to the workspace object as bellow, making workspace
unavailable:
```yaml
  status:
    conditions:
    - lastTransitionTime: "2024-04-28T10:43:29Z"
      status: "True"
      type: MountReady
    - lastTransitionTime: "2024-04-28T10:40:25Z"
      status: "True"
      type: WorkspaceScheduled
	- lastTransitionTime: "2024-04-28T10:43:29Z"
	  status: "False"
	  type: MountReady
    phase: Unavailable
```

Proposal is to make workspace conditions propagated as aggregate to workspace `phase`.

Current flow: `Creating -> Initializing -> Ready` where once workspace is created and ready
it is marked as ready and never updated.

Proposed flow: `Creating -> Initializing -> Ready <-> Unavailable.` where if any of the conditions
are not `Ready` the workspace is marked as `Unavailable`.

This would be achieved my adding new controller into `kcp` core to reconcile workspace phase
based on its own conditions.

Existing mount controller would be responsible for propagating conditions from the mount object
to the workspace object.

#### Suggested action items

1. Define the scope of the mount functionality.
2. Determine the implementation approach for the mount functionality.

#### Suggested timeline

Efforts will proceed on a best-effort basis.
