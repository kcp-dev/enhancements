# Workspace mounts enhancement proposal

## Summary

To extend spectrum of use cases for kcp, we need to support workspaces that are not
backed by a kcp cluster itself, but are backed by a mount that can be used to proxy
request to remote cluster directly or other workspaces. This enables use cases where
we can start adding external & internal integration into same workspace structure.

## Motivation

This would enable usecases where kcp is used as a proxy to remote clusters, but
some of the infrastructure is still managed by kcp. For example, kcp could be
responsible for ArgoCD, Crossplane, or other infrastructure that is more crd-based.

At this point in time communication between kcp and remote clusters is done via
workspaces abstractions.

In example:
```
$ kubectl ws tree
root:
root
  ├── clusters (workspace)
  │   ├── cluster-proxy (proxy object but shown as a workspace in tree)
  │   ├── cluster-1 (workspace)
  ├── infra (workspace)
  │   ├── argocd (workspace)
  │   ├── crossplane (workspace)
  ...
```

Where `cluster-mount` is a workspace that is backed by a mount that can be used
to proxy requests to remote clusters. This way we have unified way of interacting
with remote clusters, but still have ability to manage some of the infrastructure
via kcp.

### Goals

1. Agree how to make mount functionality as pluggable as possible.
2. Agree on the scope of the mount functionality.
3. Agree on the implementation of the mount functionality.

### Non-Goals

Proxy/Mount functionality is not meant to be a replacement for the kcp clusters.
In addition implementation of the mount functionality itself is out of scope of this
proposal. We should enable the mount functionality to be implemented by 3rd party
using kcp as a platform.

## Proposal

Introduce experimental `Mount` API to enable 2 use cases:

1. Mounting remote clusters as workspaces
2. Softlink - mounting other workspaces as sub-workspaces, where changing the sub-workspace
   would change the parent workspace as well.

### Mount API

```go

// Mount is a workspace mount that can be used to mount a workspace into another workspace or resource.
// Mounting itself is done at front proxy level.
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

	// URL is the URL of the mount. Mount is considered mountable when URL is set.
	// +optional
	URL string `json:"url,omitempty"`
}
```

Where `MountType` is an enum that would allow to specify the type of the mount:
- `URL` - mount is backed by a URL and should be used as a proxy to remote cluster.
- `Redirect` - mount is backed by another workspace and should be used as redirect to another workspace.
- `Workspace` - mount is backed by another workspace and should be used as a proxy to another workspace.

`URL` implementation should be done outside of the kcp core, but we should provide
via VirtualWorkspace.

`Redirect` and `Workspace` should be implemented in the kcp core.

`Reference` is a reference to the object that is mounted. It can be a reference to
external object used to back the mount.

Object reference would be read by KCP core mount controller and update the status
of the workspace. This way 3rd party components never need to interact with the
kcp core or workspace directly in the write mode.

```yaml
apiVersion: proxy.kcp.io/v1alpha1
kind: WorkspaceProxy
metadata:
   name: cluster-proxy
spec:
   type: passthrough
```

#### Suggested action items

1. Agree on the scope of the mount functionality.
2. Agree on the implementation of the mount functionality.

#### Suggested timeline

At the best effort basis.
