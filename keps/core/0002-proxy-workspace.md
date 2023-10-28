# Proxy workspace enhancement proposal

## Summary

To extend spectrum of use cases for kcp, we need to support workspaces that are not
backed by a kcp cluster itself, but are backed by a proxy that can be used to proxy
request to remote cluster directly, without having to go through kcp workspace.

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
  ├── tmc (tmc workspace)
  │   ├── cluster1 (TMC workspace)
  ...
```

Where `cluster-proxy` is a workspace that is backed by a proxy that can be used
to proxy requests to remote clusters. This way we have unified way of interacting
with remote clusters, but still have ability to manage some of the infrastructure
via kcp.

### Goals

1. Agree how to make proxy functionality as pluggable as possible.
2. Agree on the scope of the proxy functionality.
3. Agree on the implementation of the proxy functionality.

### Non-Goals

N/A

## Proposal

Both options below would require a new `VirtualWorkspace` implementation that
would be able to proxy requests to and from remote clusters. It would be in the separate
`go module` to avoid introducing new dependencies to the core kcp and for easier split
of the codebase.

### Option 1: Virtual workspace as a proxy

Create a `VirtualWorkspace` implementation that would be able to proxy requests
to and from remote clusters. This would be very similar to previous TMC syncer
reverse tunnels that were used to proxy requests to remote clusters.

Remote cluster would have an agent that would be responsible for establishing
a connection to the kcp cluster and would be responsible for proxying requests
to and from the remote cluster.

`https://192.168.1.136:6443/services/clusterproxy/root/cluster.proxy.kcp.io`

Proxy itself would be represented in a workspace via Cluster wide resource:
```
apiVersion: proxy.kcp.io/v1alpha1
kind: ClusterProxy
metadata:
   name: cluster-proxy
spec:
   ...<TBC>...
  status:
    conditions:
    - lastTransitionTime: "2023-10-07T15:30:12Z"
      status: "True"
      type: IdentityValid
    - lastTransitionTime: "2023-10-07T15:30:12Z"
      status: "True"
      type: VirtualWorkspaceURLsReady
    virtualWorkspaces:
    - url: https://192.168.1.138:6443/services/clusterproxy/root/cluster.proxy.kcp.io
```

Once this resource is created, it would be possible to create a pass-through
connection. Where `ws use` could detect presence of the proxy and use it
instead of the workspace itself. This would need to land somehow in the kcp core,
similar how `homeWorkspaces` are handled.

This basically removed users ability to manipulate the workspace directly, but
we could still allow users to manipulate it via custom header allowing interact with the workspace.

## Option 2: Sub-workspace

Create a fake workspace view in the CLI. This would follow same `VirtualWorkspace`
implementation as above, but would be represented as a sub-workspace. This would
show as bellow:
```
$ kubectl ws tree
root
  ├── clusters (workspace)
  │   ├── cluster-proxy (proxy object but shown as a workspace in tree)
```

This way user can still manipulate the workspace directly, but would be able to
use it as a proxy as well by: `ws use root:clusters:cluster-proxy`.

This would require some changes in the CLI to be able to detect that the workspace proxy
object is present and use it instead of the workspace itself.

#### Suggested action items

1. Agree on the possible implementation options.
2. TBC


#### Suggested timeline

Completed in: EOF Q4 2024
