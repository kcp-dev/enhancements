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
  │   ├── cluster-mount (normal workspace, with mounted cluster ontop of it)
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

1. Mounting remote clusters as workspaces
2. Softlink - mounting other workspaces as sub-workspaces, where changing the sub-workspace
would change the parent workspace as well.

### Non-Goals

* Proxy/Mount functionality is not meant to be a replacement for the kcp clusters.
In addition implementation of the mount functionality itself is out of scope of this
proposal. We should enable the mount functionality to be implemented by 3rd party
using kcp as a platform.

* Mount resources. This proposal is about enabling the mount functionality for all
workspaces, not about mounting individual resources inside workspace.

* AuthN/authZ is out of scope of this proposal. We assume that the mount functionality
implements just forwarding and redirections. Authentification and authorization
is implementers responsibility by either federating identities or using other means
to authenticate and authorize requests.

## Proposal

Introduce experimental `Mount` API to enable 2 use cases:

1. Mounting remote clusters as workspaces
2. Softlink - mounting other workspaces as sub-workspaces, where changing the sub-workspace
   would change the parent workspace as well.

Initially we would put `Mount` into annotation of the workspace, but in the future
we will promote it to spec of the workspace.

### Mount API

This is a proposal for mount API. This contains 2 parts:

1. Experimental mount API used in the annotation of the workspace.
2. Proposal how this could looks like when promoted to workspace structure.


#### Experimental mount API

As experimental API we would put mount into annotation of the workspace.

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

	// URL is the URL of the mount. Mount is considered mounted when URL is set.
	// +optional
	URL string `json:"url,omitempty"`
}
```

Where `MountType` is an enum that would allow to specify the type of the mount:
- `Proxy` - mount is backed by a URL and should be used as a proxy to remote cluster.
- `Redirect` - mount is backed by another workspace and should be used as redirect to another workspace.
- `LogicalCluster` - mount is backed by another workspace and should be used as a proxy to another workspace/logicalcluster.

`Proxy` most simple proxy implementation is `URL` where we would just proxy all requests. More advanced
implementation could include `VirtualWorkspace` or external service. But from KCP perspective
its reverse proxy to configured URL.

`Redirect` and `LogicalCluster` has two main behaviours. These include softlinking of the (external or internal).

1. Case 1: Local redirect:

As a user I want to redirect workspace `:users:john` to `:my-org:dev:john` so that I can have
a redirect inside the same instance of kcp. In this case frontoproxy returns `301 Redirect` request
with local (hostless) path to the new workspace. CLI should be able to follow the redirect
and update local kubeconfig to point to the new workspace.

2. Case 2: External redirect:

As a user I want to redirect workspace `:users:john` to `https://kcp.mycompany.com/workspaces/my-org/dev/john`
so that I can have a redirect to another kcp instance. In this case frontoproxy returns `301 Redirect` request
with full URL to the new workspace. CLI should be able to follow the redirect and update local kubeconfig.
This would allow nest different kcp instances.

`Reference` is a reference to the object that is mounted. It can be a reference to
external object used to back the mount.

Object reference would be read by KCP core mount controller. It will retrieve the
proxy URL from the referenced object's status and copy it into the mount status.
This way 3rd party components never need to interact with the kcp core or workspace
directly in write mode. Moreover, the front-proxy interpreting the mount status
URL does not have to read these mount implementation objects directly either.

```yaml
apiVersion: proxy.contrib.kcp.io/v1alpha1
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
