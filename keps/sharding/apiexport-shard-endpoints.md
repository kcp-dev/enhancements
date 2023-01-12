# APIExport evolution for sharding

## Overview

This proposal provides an API that allows controllers of service providers to retrieve the endpoints of their APIExport services across multiple shards.

It also provides an API that allows service providers to create shard Partitions:
- for the placement of controllers
- for the processing of a subset of the endpoints

## Motivation

APIExports have currently limited support for controller managers looking after multiple shards.
Service providers are facing two challenges with the current state of affairs:
- They don't know the shard topology and as such cannot take any informed decision for the scheduling of their controllers.
- There is no mechanism available for shard partitioning they could leverage to
  - have geo proximity: controllers should communicate with API servers of shards in the same region
  - scalability and load distribution: adapt the number of deployment/processes to the load pattern specific to their controllers

### Goals

1. allow controller managers to retrieve the list of endpoints for their APIExport service across multiple shards without a priori knowledge of the shards
2. allow service providers to partition the endpoints according to Shard labels, e.g. region, cloud
3. no direct access to any other shard than the one the controller has access to through its kubeconfig should be needed by service provider controllers for endpoint discovery
4. no access to cache servers for retrieving APIExports or shard URLs should bee needed by service provider controllers

### Non-Goal

1. To be 100% compatible with old APIs.
2. To automate the scheduling of controllers. This is desirable and can be covered by a higher level mechanism leveraging the proposed API but out of scope of this proposal.
3. In a multi-shards world leader election should be done per shard. This is however out of the scope of this proposal.

## Proposal

1. Introduce a new resource type **`APIExportEndpointSlice`** part of the `apis` group that acts as a sink for the endpoints of a specific APIExport. Introducing a new resource, that can be created in a workspace the service provider controllers have access to, avoids the need for the controllers to query a global resource, APIExport, through a cache server or a similar mechanism. This will technically replace `status.virtualWorkspaces.url` in APIExport making the VirtualWorkspace structure redundant. It will get deprecated as part of the implementation of this proposal.

Here is a sketch of this new resource type.
~~~
kind: APIExportEndpointSlice
apiVersion: apis.kcp.io/v1alpha1
metadata:
    name: foo-eu-west-1-asf6
spec:
    export:
        path: root:service-provider:widget
        name: widget
    # optional
    partition:  eu-west-1
status:
    conditions:
    - lastTransitionTime: "2023-01-11T14:46:19Z"
      status: "True"
      type: APIExportValid
    - lastTransitionTime: "2023-01-11T14:46:19Z"
      status: "True"
      type: EndpointURLsReady
    - lastTransitionTime: "2023-01-12T10:49:09Z"
      status: "True"
      type: PartitionValid
    endpoints:
    - url: https://192.168.1.70:6443/services/apiexport/root:service-provider/widget
    - url: https://192.168.1.71:6443/services/apiexport/root:service-provider/widget
    - url: https://192.168.1.72:6443/services/apiexport/root:service-provider/widget
~~~
A struct is used for endpoints rather than just a list of strings so that more information can be added in the future (as it was for the previous APIExport status).
The partition is made optional. If no partition is provided URLs for all shards are returned.
Similar resource types may be created for SyncTargets and ClusterWorkspaceTypes in the future. Although they will be separate APIs, these APIs will have a common pattern and their system controllers may have a significant part of the implementation in common.

2. Introduce a new resource type **`Partition`** part of a new group `topology.kcp.io`.

 Partitions can be used by the service provider:
- To create or select workspaces for their controllers in the “right” location, e.g cloud: gcp, region: europe, to have for instance geo proximity with the API servers of the shard. Therefore the information in the Partition object can be used when creating sub-workspaces. The WorkspaceSpec struct has a field [Location](https://github.com/kcp-dev/kcp/blob/9942bb0dd728045e9c61f407b3da656d1d5ae720/pkg/apis/tenancy/v1alpha1/types_workspace.go#L148-L153), which accepts a label selector to influence placement.
- To have APIExportEndpointSlice populated with a subset of the endpoints specific to the Partition. Therefore the service provider can create and reference a local Partition in the spec of the APIExportEndpointSlice resource.
The selection of shards in a `Partition` is done based on [Kubernetes label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors), where the specified selectors need to match the labels configured on the `Shards`.

~~~
kind: Partition
apiVersion: topology.kcp.io/v1alpha1
metadata:
    name: gcp-eu-all
    ownerReferences: [...]
spec:
    selector:
     matchLabels:
       cloud: gcp
       region: europe
     matchExpressions:
     - key: country
       operator: NotIn
       values:
       - KP
~~~

3. Introduce a new resource type **`PartitionSet`** part of the new group `topology.kcp.io`. PartitionSet can be used to get `Partitions` automatically created based on dimensions that match the shard label keys. The `Partitions` are created in the same workspace as `PartitionSet`.

~~~
kind: PartitionSet
apiVersion: topology.kcp.io/v1alpha1
metadata:
    name: foo
spec:
   dimensions:
   - region
   - cloud
   selectors:
     matchExpressions:
     - key: region
       operator: NotIn
       values:
       - Antarctica
       - Greenland
     - key: country
       operator: NotIn
       values:
       - NK
status:
   count: 10
~~~

With 2 dimensions: regions and clouds, if we have n regions and m clouds up to n x m Partitions will be created. If there is no shard in an intersection, e.g. cloud: azure, region: antarctica, no Partition resource gets created for it so that there cannot be more Partitions than the total number of shards.

### Consequences

- Existing external controllers, kcp controller-runtime fork, the controller-runtime example and the SDK prototype will need to be amended to use `APIExportEndpointSlice` instead of `APIExport`.
- Controller managers should spawn a new informer / controller for each shard in its scope. They should reconcile the informers with the list of shards, whenever it get modified.
- Shards added to an existing partition may get automatically picked up by a controller manager. It may not be the case for shards that are added to a new topology label, e.g. when a new region gets added.

## Implementation

A phased approach can be taken for the implementation. The controller for APIExportEndpointSlice will be implemented first as it will be enough:
- for enabling service providers to write multi-shard capable controllers making use of the new API
- for enhancing the client side tooling for multi-shard capable controllers

Next the processing of Partitions will follow and the implementation of the controller for PartitioSet will come last.

## TODO/Question

- In the future we could consider whether we want to add support for `self` in Partitions. An approach would be to have it as a reserved word for label values `region: self`. Another approach would be to move away from the traditional label selector implementation and add a `self` boolean field.
~~~
selector:
    - name: cloud
      value: gcp
    - name: region
      self: true
~~~

## Alternatives

See [the Google doc](https://docs.google.com/document/d/1zI9KOpLp6H_tAcKpGbLpQu_iUK9VbpgUSo0OFEd7Mig/edit#), where the design and possible alternatives have been discussed.
