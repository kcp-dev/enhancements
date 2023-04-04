# KEP-0002: Model Entitlements for Cross-Workspace Interactions

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [System Administrators Can Configure External Entitlement Checks](#system-administrators-can-configure-external-entitlement-checks)
    - [Service Providers On `kcp` Can Gate Access To Their Service](#service-providers-on-kcp-can-gate-access-to-their-service)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Summary

This KEP describes an API that would allow users of `kcp` to predicate use of
their cross-workspace relationships on an entitlement system. Furthermore, this
KEP introduces a provisional implementation of the API that will suffice for
bootstrapping and small-scale environments.

## Motivation

`kcp` distinguishes itself from other Kubernetes-based control planes by providing
not only cheap multi-tenancy through workspaces but also allowing users to model
cross-workspace concerns. While a fleet of entirely independent workspaces has
value, it's these cross-workspace interactions that are uniquely possible in `kcp`.
One common type of cross-workspace interaction is a provider/consumer split - one
actor provides some service through content in their workspace, and others using
the system can consume that service. Examples of this relationship include:

 - `APIExports`, used via `APIBindings` from the `apis.kcp.io` group
 - `WorkspaceTypes`, used in creating `Workspaces` from the `tenancy.kcp.io` group
 - `Locations`, used via `Placements` from the `scheduling.kcp.io` group

Naturally, providers of these functionalities would like to manage where their services
can be used. Today, restricting access to the provider's service is done in an ad-hoc
manner (or not implemented at all!), for example with `APIExports` the flow looks like:

 - create RBAC in the `APIExport` workspace that provides `verb="bind"` access to the
   particular `APIExport` name in question
 - gate access to the cross-workspace API by implementing a validating admission plugin
   that issues a `SubjectAccessReview` using a "deep" client that skips other authorizers
   while issuing the SAR

Notably, use of the "deep" `SubjectAccessReview` client requires an admission _plugin_,
and is privileged. Therefore, the bar for implementing new cross-workspace concepts
with any sort of authorization is high. Even for in-tree, privileged APIs, this is a
non-trivial implementation cost.

This KEP proposes a mechanism to model entitlements to cross-workspace concepts both for
in-tree uses as well as user-provided extensions.

### Goals

1. Model entitlements to cross-workspace concerns in a generic manner
1. Enable entitlement checking in a non-privileged manner
1. Provide an implementation for entitlement checking that suffices for bootstrapping
1. Enable users to plug in custom, scalable implementations trivially
1. Simplify the `kcp` authorizer chain by removing non-Kubernetes, non-RBAC concepts like
   the workspace authorizer and "deep" SubjectAccessReviews

### Non-Goals

1. Enforce a particular data model or implementation
1. Provide an implementation that scales well
1. Remove all custom RBAC objects from `kcp` systems

## Proposal

`kcp` will expose a new API for reviewing entitlements, the implementation of which
will be configured at server start-up. This endpoint will act as a well-known proxy
to the implementation being used, in order to allow downstream consumers to be
unaware of changes to the implementation. Access to this API will be done through
workspace-specific endpoints and authorized on a per-endpoint basis. Users building
cross-workspace concepts on top of `kcp` will be able to write admission _webhooks_
that use this API to implement restrictions to the use of their concepts.

Additionally, a default implemenation will be provided for `kcp start` to allow for
bootstrappiung the system. This implemenation will be self-sufficient at small scales,
but the expectation will be that users with more complex entitlement models will use
third-party systems based on more performant databases to implement their entitlement
reviews. 

### User Stories

#### System Administrators Can Configure External Entitlement Checks

A system administrator is using `kcp` to provide a multi-tenant platform for developers
at Acme Corp. Internally, Acme Corp. records which developers or teams have access to
which services. By implementing a delegate entitlement review webhook, the system
administrator or platform team can connect `kcp` entitlement checks to the external
system of record.

#### Service Providers On `kcp` Can Gate Access To Their Service

A team at Acme Corp invents a new cross-workspace paradigm whereby consumers enable
their workspaces to make use of the paradigm. In order to gate access to this new
functionality, the provider team writes a validating admission webhook and uses the
standard `kcp` entitlement review API to ensure only authorized use of their paradigm
is allowed.

### Notes/Constraints/Caveats

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above?
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

## Design Details

A new endpoint will be exposed to facilitate entitlement review. Users will `POST`
an `EntitlementReview` object to the endpoint and synchronously receive the result
in the response. The `EntitlementReview` has the following form:

```go
type EntitlementReview struct {
    metav1.TypeMeta `json:",inline"`
    // Spec holds information about the request being evaluated.
    Spec EntitlementReviewSpec `json:"spec"`

    // Status is filled in by the server and indicates whether the request is allowed or not.
    // +optional
    Status EntitlementReviewStatus `json:"status,omitempty"`
}

// EntitlementReviewSpec is a description of the entitlement request.
type EntitlementReviewSpec struct {
    // UserInfo is information about the requesting user
    UserInfo authenticationv1.UserInfo `json:"userInfo"`

    // RequestInfo is information about the context in which this request was issued.
    RequestInfo RequestInfo `json:"requestInfo"`

    // Entitlement is the entitlement to be reviewed.
    Entitlement Entitlement `json:"entitlement"`
}

// RequestInfo is a description of the context in which a request is issued.
type RequestInfo struct {
    // ClusterName is the name of the logical cluster in which the request was issued.
    ClusterName logicalcluster.Name `json:"clusterName"`

    // ClusterPath is the human-readable path to the logical cluster in which the request was issued.
    ClusterPath logicalcluster.Path `json:"clusterPath"`
}

// Entitlement describes an entitlement
type Entitlement struct {
    // ClusterName is the name of the logical cluster from which the entitlement is provided.
    // NOTE: the entitlement review API is segmented by this cluster name in order to allow
    // fine-grained authorization for the review API, so this field is superfluous; however, 
    // duplicating the data here allows for the entitlement review object to be self-sufficient.
    // +optional
    ClusterName logicalcluster.Name `json:"clusterName"`

    // The type of entitlement being described below is free-form data to be interpreted by
    // the implementor of the review server. This type metadata allows for the object to be
    // implemented as a discriminated union.
    metav1.TypeMeta `json:",inline"`

    // Spec is the free-form entitlement to be reviewed.
    Spec runtime.RawExtension `json:"spec,omitempty"`
}

// EntitlementReviewStatus contains the result of the entitlement review.
type EntitlementReviewStatus struct {
    // Entitled is required. True if the entitlement is granted, false otherwise.
    Entitled bool `json:"entitled"`
    // Disqualified is optional. True if the entitlement is not granted, otherwise
    // false. If both entitled is false and disqualified is false, then the
    // reviewer has no opinion on whether to grant the entitlement. Disqualified
    // may not be true if entitled is true.
    // +optional
    Disqualified bool `json:"disqualified,omitempty"`
    // Reason is optional. It indicates why a request was entitled or disqualified.
    // +optional
    Reason string `json:"reason,omitempty"`
    // EvaluationError is an indication that some error occurred during the entitlement review.
    // It is entirely possible to get an error and be able to continue determine entitlement status in spite of it.
    // +optional
    EvaluationError string `json:"evaluationError,omitempty"`
}
```

The entitlement review API will be partitioned by workspace; entitlement reviews will be issued
to the workspace-specific endpoint relevant to the entitlement and access to this API will be 
guarded by Kubernetes RBAC in the workspace:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: core:entitlements:review
rules:
- apiGroups: ["core.kcp.io"]
  resources:
  - "entitlementreviews"
  verbs: ["create"]
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: core:entitlements:review
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: core:entitlements:review
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:authenticated
```

Use of the entitlement review API might look like

```
> POST /services/entitlementreview/clusters/<entitlement-provider-cluster>/apis/core.kcp.io/v1alpha1/entitlementreviews
> {
>     "kind": "EntitlementReview",
>     "apiVersion": "core.kcp.io/v1alpha1",
>     "spec": {
>         "userInfo": {
>             "username": "admin",
>             "uid": "014fbff9a07c",
>             "groups:": [
>             - "system:authenticated",
>             - "some-custom-group"
>             ],
>             "extra": {
>                 "some-key": [
>                 - "some-value",
>                 - "some-other-value"
>                 ]
>             }
>         },
>         "requestInfo": {
>             "clusterName": "f5865fce",
>             "clusterPath": "root:management:us-west-invoices"
>         },
>         "entitlement": {
>             "clusterName": "33bab531",
>             "kind": "WidgetAccess",
>             "apiVersion": "acme.corp/v1",
>             "spec": {
>                 "something": {
>                     "meaningful": "here"
>                 }
>             }
>         }
>     }
> }
200 OK
{
    "kind": "EntitlementReview",
    "apiVersion": "core.kcp.io/v1alpha1",
    "spec": {
        "userInfo": {
            "username": "admin",
            "uid": "014fbff9a07c",
            "groups:": [
            - "system:authenticated",
            - "some-custom-group"
            ],
            "extra": {
                "some-key": [
                - "some-value",
                - "some-other-value"
                ]
            }
        },
        "requestInfo": {
            "clusterName": "f5865fce",
            "clusterPath": "root:management:us-west-invoices"
        },
        "entitlement": {
            "clusterName": "33bab531",
            "kind": "WidgetAccess",
            "apiVersion": "acme.corp/v1",
            "spec": {
                "something": {
                    "meaningful": "here"
                }
            }
        }
    },
    "status": {
        "entitled": true,
        "disqualified": false,
        "reason": "All users in `root:management:us-west-invoices` are entitled to `meaningful:here`."
    }
}
```

### Tenancy-Based Implementation

In addition to providing the abstract concept of and endpoints for entitlement review, `kcp`
will also provide a simple implementation tied to workspaces as tenancy boundaries. This
implementation is intended to be sufficient for use-cases where the complexity of entitlements
is low, the rate of change is sufficiently low and the incidence of strict entitlement rules
is rare. This implementation will make use of two objects in the `entitlements.tenancy.kcp.io` group:

```go
type EntitlementPolicy struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // Entitlements holds all the entitlements that this EntitlementPolicy
    // groups. 
    // +optional
    Entitlements []kcpcorev1alpha1.Entitlement `json:"entitlements"`
}

type EntitlementPolicyBinding struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    // EntitlementPolicyRef is a reference to the entitlement policy
    // that this binding applies to.
    EntitlementPolicyRef EntitlementPolicyRef `json:"entitlementPolicyRef"`

    // Children determines if this policy binding provides the entitlement just to the
    // workspace in which it is created, or also to the child workspaces underneath that.
    // Optional, defaults to false.
    // +optional
    Children boolean `json:"children,omitempty"`
}

type EntitlementPolicyRef struct {
    // ClusterPath is the human-readable path to the logical cluster in which the entitlement
    // policy is registered.
    ClusterPath logicalcluster.Path `json:"clusterPath"`

    // Name is the name of the entitlement policy.
    Name string `json:"name"`
}
```

A service provider with permissions to create `EntitlementPolicy` will do so using the
normal `kcp` API:

```
> POST /clusters/<entitlement-provider-cluster>/apis/entitlements.tenancy.kcp.io/v1alpha1/entitlementpolicies
> {
>     "kind": "EntitlementPolicy",
>     "apiVersion": "entitlements.tenancy.kcp.io/v1alpha1",
>     "metadata": {
>         "name": "something-meaningful"
>     },
>     "entitlements": [
>         {
>             "clusterName": "33bab531",
>             "kind": "WidgetAccess",
>             "apiVersion": "acme.corp/v1",
>             "spec": {
>                 "something": {
>                     "meaningful": "here"
>                 }
>             }
>         }
>     ]
> }
201 CREATED
{
    "kind": "EntitlementPolicy",
    "apiVersion": "entitlements.tenancy.kcp.io/v1alpha1",
    "metadata": {
        "name": "something-meaningful"
    },
    "entitlements": [
        {
            "clusterName": "33bab531",
            "kind": "WidgetAccess",
            "apiVersion": "acme.corp/v1",
            "spec": {
                "something": {
                    "meaningful": "here"
                }
            }
        }
    ]
}
```

However, to entitle tenants, a per-tenant endpoint is used:

```
> POST /services/entitlements/clusters/<entitlement-consumer-cluster>/apis/entitlements.tenancy.kcp.io/v1alpha1/entitlementpolicybindings
> {
>     "kind": "EntitlementPolicyBinding",
>     "apiVersion": "entitlements.tenancy.kcp.io/v1alpha1",
>     "metadata": {
>         "name": "something-meaningful-for-a-tenant"
>     },
>     "entitlementPolicyRef": {
>         "clusterPath": "<entitlement-provider-cluster>",
>         "name": "something-meaningful"
>     },
>     children: false
> }
201 CREATED
{
    "kind": "EntitlementPolicyBinding",
    "apiVersion": "entitlements.tenancy.kcp.io/v1alpha1",
    "metadata": {
        "name": "something-meaningful-for-a-tenant"
    },
    "entitlementPolicyRef": {
        "clusterPath": "<entitlement-provider-cluster>",
        "name": "something-meaningful"
    },
    children: false
}
```

Entitlement policy bindings will be stored in tenant clusters; however, the tenant will have
no access to read or change them. This mechanism is used to distribute this data, reduce its
cardinality and remove the need to cache policy bindings that do not implicate children. For
those entitlement policy bindings which _do_ set `"children":true`, replication using the cache
server will be required.

A server will also exist that can field entitlement review requests. This server will entirely
ignore the `userInfo` section of the entitlement review request, and only operate on the request
information. A simple deep equality will be done to determine if an entitlement in the request
matches one in a policy.

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[ ] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit

This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->

- `<package>`: `<date>` - `<test coverage>`

##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

- <test>: <link to test coverage>

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

- <test>: <link to test coverage>

### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, [feature gate] graduations, or as
something else. The KEP should keep this high-level with a focus on what
signals will be looked at to determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Feature gate][feature gate] lifecycle
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
or by redefining what graduation means.

In general we try to use the same stages (alpha, beta, GA), regardless of how the
functionality is accessed.

[feature gate]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled

#### Beta

- Gather feedback from developers and surveys
- Complete features A, B, C
- Additional tests are in Testgrid and linked in KEP

#### GA

- N examples of real-world usage
- N installs
- More rigorous forms of testing—e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least two releases between beta and
GA/stable, because there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

**For non-optional features moving to GA, the graduation criteria must include
[conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md

#### Deprecation

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag
-->

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.
-->

### Feature Enablement and Rollback

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

<!--
Pick one of these and delete the rest.

Documentation is available on [feature gate lifecycle] and expectations, as
well as the [existing list] of feature gates.

[feature gate lifecycle]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[existing list]: https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/
-->

- [ ] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name:
  - Components depending on the feature gate:
- [ ] Other
  - Describe the mechanism:
  - Will enabling / disabling the feature require downtime of the control
    plane?
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?

###### Does enabling the feature change any default behavior?

<!--
Any change of default behavior may be surprising to users or break existing
automations, so be extremely careful here.
-->

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

<!--
Describe the consequences on existing workloads (e.g., if this is a runtime
feature, can it break the existing applications?).

Feature gates are typically disabled by setting the flag to `false` and
restarting the component. No other changes should be necessary to disable the
feature.

NOTE: Also set `disable-supported` to `true` or `false` in `kep.yaml`.
-->

###### What happens if we reenable the feature if it was previously rolled back?

###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.

Additionally, for features that are introducing a new API field, unit tests that
are exercising the `switch` of feature gate itself (what happens if I disable a
feature gate after having objects written with the new field) are also critical.
You can take a look at one potential example of such test in:
https://github.com/kubernetes/kubernetes/pull/97058/files#diff-7826f7adbc1996a05ab52e3f5f02429e94b68ce6bce0dc534d1be636154fded3R246-R282
-->

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [ ] Events
  - Event Reason: 
- [ ] API .status
  - Condition name: 
  - Other field: 
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases
(e.g. probes taking a minute instead of milliseconds, failed pods consuming resources, etc.).
If any of the resources can be exhausted, how this is mitigated with the existing limits
(e.g. pods per node) or new limits added by this KEP?

Are there any tests that were run/should be run to understand performance characteristics better
and validate the declared limits?
-->

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->
