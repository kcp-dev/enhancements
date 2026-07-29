# Ephemeral Resources Enhancement Proposal

## Summary

kcp today can serve an API only if it can store it. Every resource exposed through an
`APIResourceSchema` is backed by etcd: a client `POST` writes an object, a controller
reconciles it, and the client polls or watches for `status`. For a whole class of
provider APIs this is the wrong shape. The client is asking a *question* and wants an
*answer*, not a record.

Kubernetes already has this shape internally. `SubjectAccessReview`, `SelfSubjectReview`
and `TokenReview` are submitted with `POST`, answered synchronously in the response body,
and never persisted. They are implemented as bespoke REST storage inside the API server,
so the pattern is unavailable to anyone extending the API surface.

This proposal introduces **ephemeral resources**: an `APIResourceSchema` may declare that
its instances are never persisted, and nominate a webhook that answers requests against
them. kcp serves the resource as a normal, discoverable, RBAC-governed API; on `create`
it forwards the submitted object to the provider's webhook and returns the webhook's
response object to the client. Nothing reaches storage.

## Motivation

The concrete case that prompted this: an operator wants a `BucketInfo` resource. A client
`POST`s `spec.bucketName` and gets back live `status.sizeBytes` and `status.objectCount`
in the response. The numbers come from an object store that already knows them
authoritatively.

Every mechanism available today is a workaround:

- **Persisted CR + controller.** The provider writes a real object per query, a controller
  fills in `status`, the client polls. This turns a read into a write, puts per-query
  garbage in etcd with a GC problem attached, and gives the client stale data by
  construction: the value in `status` is whatever the last reconcile saw. For a metric
  like bucket size, that is worse than useless.
- **Workspace mounts.** kcp has no aggregation layer. There is no
  `apiregistration.k8s.io` group and no `APIService`, and the model does not admit one,
  since a logical cluster has no in-cluster network to proxy into. The nearest equivalent is
  `Workspace.spec.mount`, which delegates a workspace to an external URL. But mounts
  operate at *workspace* granularity: the provider must take over the entire workspace to
  serve one resource, and loses every other API in it. That is the wrong unit for
  "return three numbers".
- **Side-channel HTTP endpoint.** Abandons the API machinery entirely: no discovery, no
  RBAC, no client-go, no `kubectl`. The provider now ships an SDK.

None of these are acceptable for a provider whose whole value proposition is *"bind my
APIExport and use `kubectl`"*.

### Precedent inside kcp

The concept is not foreign to the codebase. `pkg/permissionclaim/permissionclaim_labeler.go`
already carries a hardcoded list of resources kcp knows are not persisted:

```go
// NonPersistedResourcesClaimable is a list of resources that are not persisted
// to etcd, and therefore should not be labeled with permission claims. The value
// means whether they are claimable or not.
var NonPersistedResourcesClaimable = map[schema.GroupResource]bool{ ... }
```

and the apiexport virtual workspace already serves `TokenReview` as a built-in
(`pkg/virtual/apiexport/schemas/builtin/builtin.go`), with the response produced by
delegated authentication rather than storage. Both establish that non-persisted resources
fit kcp's model. What is missing is a way for a *provider* to declare one.

### Goals

1. Allow an `APIResourceSchema` to declare a resource as ephemeral: served, discoverable,
   RBAC-governed, never written to etcd.
2. Define a webhook contract by which kcp obtains the response object, modelled on the
   existing admission and conversion webhook contracts.
3. Work identically for consumers via `APIBinding` and for providers via the apiexport
   virtual workspace.
4. Keep the request stateless, so that any shard can serve it without coordination.

### Non-Goals

1. **No `list` or `watch`.** Ephemeral resources have no collection. `list` returns an
   empty list, `watch` is not served. Providers wanting collections should persist.
2. **`create` is the only verb, permanently.** There is nothing to mutate or remove, and
   `get` is not a deferred feature but a closed question. An ephemeral resource's
   query lives in a custom `spec`, and there is no safe way to put an arbitrary `spec` in
   a URL (see "Why `get` cannot work" below).
3. **No caching, deduplication, or retry by kcp.** The webhook is authoritative on every
   request; retries are the client's business.
4. **kcp does not run the webhook.** Deployment, scaling and availability of the webhook
   backend are the provider's responsibility, exactly as with admission webhooks.
5. **No change to how persisted resources behave.** A schema without `spec.ephemeral` is
   unaffected.

## Proposal

### 1. Declaring an ephemeral resource

Two things need expressing, and they have different lifetimes:

- **That the resource is non-persisted.** This is an API contract property. It belongs on
  the `APIResourceSchema` and is correctly immutable: a resource cannot stop being
  ephemeral without becoming a different API.
- **Where the webhook is and how to authenticate to it.** This is deployment
  configuration. Endpoints move, CAs expire, client certs rotate. It must be mutable.

`APIResourceSchema.spec` is immutable in its entirety, since
`ValidateAPIResourceSchemaUpdate` rejects any spec change with `"is immutable"`. Putting
a `caBundle` or a URL there would make CA rotation impossible without minting a new schema
and repointing the APIExport. That is a disruptive operation for consumers, and it would
break automated CA injection outright: tools like cert-manager's `ca-injector`
work by *writing* the bundle into the object, which an immutable spec forbids.

So the schema carries only a marker:

```yaml
apiVersion: apis.kcp.io/v1alpha1
kind: APIResourceSchema
metadata:
  name: v1alpha1.bucketinfos.s3.example.com
spec:
  group: s3.example.com
  scope: Namespaced
  names:
    plural: bucketinfos
    singular: bucketinfo
    kind: BucketInfo
  versions:
  - name: v1alpha1
    served: true
    storage: false          # see validation rules below
    schema: { ... }
  ephemeral: {}             # marker: instances are never persisted
```

and the `APIExport`, which is mutable, lives in the provider's workspace, and already
holds the identity secret reference, carries the endpoint:

```yaml
apiVersion: apis.kcp.io/v1alpha1
kind: APIExport
metadata:
  name: s3.example.com
spec:
  resources:
  - name: bucketinfos
    group: s3.example.com
    schema: v1alpha1.bucketinfos.s3.example.com
  ephemeralEndpoints:
  - group: s3.example.com
    resource: bucketinfos
    url: https://s3-info.example.com/ephemeral/bucketinfos
    caBundleRef:                       # server trust: ConfigMap, CA-injector friendly
      namespace: kcp-system
      name: s3-info-ca
      key: ca.crt
    timeoutSeconds: 10
    failurePolicy: Fail
status:
  ephemeralEndpoints:
  - group: s3.example.com
    resource: bucketinfos
    caBundle: <resolved bundle, replicated to all shards>
```

The reference resolves in the **provider's workspace**, matching how
`spec.identity.secretRef` already works, including in how it is *consumed*, which the
next section covers. Rotation is a normal ConfigMap update: no schema churn, no consumer
impact.

There is deliberately no field for a client certificate. See below.

Validation rules:

- `spec.ephemeral` and `spec.conversion` are mutually exclusive. There is one wire
  version's worth of object and no stored version to convert from.
- Today exactly one version must have `storage: true`. For ephemeral schemas this is
  relaxed: no version may set `storage: true`, since nothing is stored. Multiple served
  versions are allowed, and the version is passed to the webhook so it can answer in the
  version requested.
- `subresources` are not permitted. The `status` of an ephemeral object is part of the
  single response, not a separately addressable endpoint, the same way
  `SelfSubjectReview` returns its `status` inline.
- Every schema marked `ephemeral` in `spec.resources` must have a matching entry in
  `spec.ephemeralEndpoints`, and vice versa. A mismatch surfaces as a condition on the
  APIExport rather than a webhook failure at request time.

### 1a. The cross-shard constraint

kcp does not replicate Secrets or ConfigMaps between shards, and this is deliberate.
`pkg/reconciler/cache/replication/replication_controller.go` enumerates exactly what the
cache server carries: `apiexports`, `apiresourceschemas`, `apiconversions`, webhook
configurations, `shards`, `logicalclusters`, `clusterroles`, all of it public API
metadata. `secrets` and `configmaps` appear nowhere in that list.

This has a direct consequence for any credential referenced from an APIExport. A logical
cluster lives on one shard, so a Secret in the provider's workspace exists only on the
shard hosting that workspace. But an ephemeral request arrives at the shard hosting the
**consumer's** workspace, which is generally a different one. That shard cannot read the
provider's Secret.

kcp already has the answer to this, in the identity mechanism: the APIExport controller
reads the identity Secret *locally*, on the shard that hosts the export, and publishes the
derived `status.identityHash`. The APIExport, status included, replicates. The secret
never crosses a shard boundary; a non-secret value derived from it does.

The same pattern applies to the CA bundle. `caBundleRef` is resolved on the provider's
shard and materialized into `status.ephemeralEndpoints[].caBundle`, which replicates
everywhere. A CA bundle is public by nature, so publishing it in status is safe. Providers
keep the CA-injector-friendly ConfigMap; every shard gets the bytes.

### 1b. Why kcp issues its own client certificate

Client authentication is not optional here. The aggregation layer mandates mutual TLS.
The kube-apiserver presents `--proxy-client-cert-file` and the extension server verifies
it against the requestheader CA, and it does so for a specific reason: the aggregated
server is being *told* who the user is and must know the assertion came from the API
server. `EphemeralReview.request.userInfo` puts an ephemeral webhook in exactly that
position. Without client authentication, anyone who can reach the endpoint asserts
arbitrary identity and reads any user's data.

What does *not* work is a provider-supplied certificate. Following the constraint above:

- **Replicating the Secret** would copy provider-held private keys into every shard's
  etcd. That inverts the reason Secrets are excluded from replication in the first place.
- **Proxying the call through the provider's shard** puts the webhook call back on the
  shard that can read the Secret. It adds a hop, needs its own shard-to-shard
  authentication, and reintroduces shard affinity for a request that has no stored state,
  discarding the main structural advantage of ephemerality (see section 3).
- **A certificate per shard, supplied by the provider** makes the provider track kcp's
  topology: issue a cert per shard, and reissue whenever a shard is added. No provider
  should need to know how many shards a kcp installation runs.

The third option also exposes the deeper point. Once the certificate is per-shard, it is
identifying *kcp*, not the provider relationship, so per-provider or per-binding
granularity buys nothing at all. There is no authorization decision a webhook can make
from "this is the cert I issued to kcp" that it cannot make from "this is kcp, verified
against kcp's published CA". The per-provider certificate is complexity with no
corresponding capability.

So kcp presents its own identity, and providers verify it against a CA bundle kcp
publishes. This is the aggregation-layer model (one API server identity, N extension
servers, CA distributed via the `extension-apiserver-authentication` ConfigMap), and it is
the model that scales, because a provider needs no PKI at all, only a bundle to trust.

It is also not new machinery. kcp already mints and presents a requestheader-CA-signed
client certificate on outbound calls in the mounts path
(`--mount-proxy-client-cert-file`, wired in `pkg/server/localproxy.go`). What is missing
is only a way to publish the corresponding CA to providers, and it should be a
**separate** CA from the front-proxy requestheader one, since a key signed by the latter
is an identity-header impersonation credential and must never be handed to a third party.
See Open Questions.

### 2. The webhook contract

kcp `POST`s an `EphemeralReview` to the webhook, modelled directly on `AdmissionReview`
so that implementers can reuse the mental model and most of the plumbing:

```yaml
apiVersion: apis.kcp.io/v1alpha1
kind: EphemeralReview
request:
  uid: <request uid>
  cluster: <logical cluster name>
  resource: {group: s3.example.com, version: v1alpha1, resource: bucketinfos}
  namespace: team-a
  userInfo: {username: ..., groups: [...], extra: {...}}
  object: <the object the client submitted>
```

and expects back:

```yaml
apiVersion: apis.kcp.io/v1alpha1
kind: EphemeralReview
response:
  uid: <echoed>
  allowed: true
  object: <the object returned to the client>
  # or, when allowed: false
  status: {code: 404, reason: NotFound, message: "bucket does not exist"}
```

Semantics:

- The returned `object` is written straight to the client as the `201` response body. kcp
  validates it against the schema and rejects a non-conforming response with `500`, so a
  broken webhook cannot smuggle arbitrary content through a typed API.
- `allowed: false` with a `status` lets the webhook return a proper API error. This is how
  a provider says "no such bucket" as a `404` rather than an opaque failure.
- `userInfo` and `cluster` are the provider's authorization inputs. kcp has already
  enforced RBAC on `create` for the group/resource; anything finer, such as *may this user
  see this bucket*, is the webhook's decision.
- `failurePolicy: Fail` returns `503` to the client when the webhook is unreachable or
  times out; `Ignore` returns the submitted object unchanged with an empty `status`.
  `Fail` is the default, since an ephemeral resource with no answer has no value.

### 2a. Why `get` cannot work

`POST` is not a stylistic choice inherited from `SelfSubjectReview`; it is the only verb
that can carry the request. A `get` would have to encode the query, an arbitrary
provider-defined `spec`, into a URL.

Kubernetes has exactly one mechanism for this, `runtime.ParameterCodec` backed by
`queryparams.Convert`, and it is not general. Reading `convertStruct` in
`k8s.io/apimachinery/pkg/conversion/queryparams`:

- primitive fields become query parameters;
- slices are encoded **only** when their element type is primitive, so a slice of
  structs falls through the switch and is silently dropped;
- maps are not handled at all and are silently dropped;
- nested structs are recursed into but flattened into one namespace with no path
  prefixing, so a nested field tag collides with a top-level one of the same name.

Data loss is silent in every one of those cases; nothing returns an error. This is why
every options type in Kubernetes (`ListOptions`, `PodLogOptions`, `PodExecOptions`) is
deliberately flat and primitive. An `APIResourceSchema` `spec` is a full OpenAPI schema
with nesting, arrays of objects, and maps, and cannot be constrained to that shape without
making it not a schema.

The alternatives are worse. Serializing the spec into a single opaque query parameter
abandons typing, discovery and validation, and runs into URL length limits. Forcing the
query into the resource *name*, as in `GET bucketinfos/my-bucket`, works only for
single-scalar queries and conflates identity with parameters, so it stops working the
moment a provider adds a second field.

So `EphemeralReview` does not carry a verb field. There is no second verb for it to
distinguish, and adding the field would imply a roadmap that does not exist.

### 2b. Assumption: auditing the answer is the provider's job

kcp audits an ephemeral request the way it audits any API request (user, verb, resource,
workspace, response code) through the existing audit chain, with no special handling
required. What it does not record is the response body, because that body is not an object
and never becomes one.

This is treated as correct rather than as a gap. The provider generates the response, and
the provider is the only party that knows what the values mean or which of them are
sensitive. Any audit trail over the *content* of an answer belongs on the provider's side
of the webhook, where the data originates and where the retention policy for it is set.
Duplicating it in kcp's audit log would mean kcp storing provider payloads it cannot
interpret, from a resource whose entire premise is that kcp stores nothing.

### 3. Where the request is served

This is the question that stalled the discussion, and ephemerality is what makes it
tractable: **because nothing is stored, no shard owns the request.** There is no storage
locality to respect and no per-shard state to reconcile. Whichever shard terminates the
client's request calls the webhook and returns the answer. The only requirement is
network reachability from shards to the webhook endpoint, the same requirement admission
webhooks already impose.

Two serving paths, one implementation:

- **Consumer path (`APIBinding`).** The bound resource appears in the consumer workspace
  as usual. Instead of the CR storage, the schema's `RestProviderFunc` returns an
  ephemeral REST storage implementing only `Creater`.
- **Provider path (apiexport virtual workspace).** The same storage is installed by the
  apireconciler, alongside the existing built-ins. This is structurally what
  kcp-dev/kcp#4280 does for `TokenReview`: schema registered, response produced off a
  non-storage path.

The existing `apiserver.RestProviderFunc` hook, already used by
`provideAPIExportFilteredRestStorage` and `provideDelegatingRestStorage` in
`pkg/virtual/apiexport/builder/forwarding.go`, is the correct insertion point for both.

### 4. Permission claims

Ephemeral resources must be excluded from permission-claim labeling for the reason the
existing comment gives: there is no object to label. The static
`NonPersistedResourcesClaimable` map becomes a fallback for the core resources it already
lists, and the labeler additionally consults the bound schema: a resource whose schema
declares `spec.ephemeral` is non-persisted and non-claimable. The check in
`permissionclaim_labeler.go` and the skip in
`pkg/reconciler/apis/permissionclaimlabel/permissionclaimlabel_reconcile.go` both need to
account for this.

Whether ephemeral resources should later become *claimable*, so an export can claim
another export's ephemeral resource and call through it, is deliberately left open. It
is additive and should follow real demand.

## API Changes

`staging/src/github.com/kcp-dev/sdk/apis/apis/v1alpha1/types_apiresourceschema.go`:

```go
// --- types_apiresourceschema.go ---

type APIResourceSchemaSpec struct {
	// ... existing fields ...

	// ephemeral marks this resource as non-persisted. Instances are never written
	// to storage; each create is answered synchronously by the endpoint configured
	// on the APIExport that exposes this schema.
	//
	// Mutually exclusive with conversion. When set, no version may set storage: true.
	// This field is part of the API contract and, like the rest of the spec, immutable.
	// Endpoint and credential configuration deliberately lives on the APIExport, which
	// is mutable.
	//
	// +optional
	Ephemeral *EphemeralResource `json:"ephemeral,omitempty"`
}

// EphemeralResource marks a resource as non-persisted. It is intentionally empty:
// everything operational belongs on the APIExport.
type EphemeralResource struct{}

// --- types_apiexport.go ---

type APIExportSpec struct {
	// ... existing fields ...

	// ephemeralEndpoints configures the webhook backing each ephemeral resource
	// exposed by this APIExport. Every schema in spec.resources whose
	// APIResourceSchema sets spec.ephemeral must have exactly one entry here.
	//
	// +optional
	// +listType=map
	// +listMapKey=group
	// +listMapKey=resource
	EphemeralEndpoints []EphemeralEndpoint `json:"ephemeralEndpoints,omitempty"`
}

type EphemeralEndpoint struct {
	// group and resource identify which ephemeral resource this endpoint answers.
	//
	// +required
	Group string `json:"group"`
	// +required
	Resource string `json:"resource"`

	// url is the HTTPS endpoint kcp calls. Plain http is rejected.
	//
	// +required
	// +kubebuilder:validation:Pattern=`^https://`
	URL string `json:"url"`

	// caBundleRef references a ConfigMap in the APIExport's workspace holding the CA
	// bundle used to verify the endpoint's serving certificate. A ConfigMap rather
	// than an inline bundle so that CA injection tooling can write to it. The bundle
	// is resolved on this workspace's shard and published to
	// status.ephemeralEndpoints[].caBundle, since ConfigMaps do not replicate between
	// shards. Defaults to the system trust store when unset.
	//
	// There is intentionally no client certificate field: kcp presents its own
	// identity and publishes the CA for providers to verify against.
	//
	// +optional
	CABundleRef *ConfigMapKeyReference `json:"caBundleRef,omitempty"`

	// timeoutSeconds bounds how long kcp waits for a response. Defaults to 10.
	//
	// +optional
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=30
	// +kubebuilder:default=10
	TimeoutSeconds int32 `json:"timeoutSeconds,omitempty"`

	// failurePolicy determines the behaviour when the webhook cannot be reached.
	// Fail returns 503 to the client; Ignore returns the submitted object unchanged.
	//
	// +optional
	// +kubebuilder:validation:Enum=Fail;Ignore
	// +kubebuilder:default=Fail
	FailurePolicy string `json:"failurePolicy,omitempty"`
}

type APIExportStatus struct {
	// ... existing fields ...

	// ephemeralEndpoints carries per-endpoint state resolved on this APIExport's
	// shard and replicated to all shards.
	//
	// +optional
	// +listType=map
	// +listMapKey=group
	// +listMapKey=resource
	EphemeralEndpoints []EphemeralEndpointStatus `json:"ephemeralEndpoints,omitempty"`
}

type EphemeralEndpointStatus struct {
	// +required
	Group string `json:"group"`
	// +required
	Resource string `json:"resource"`

	// caBundle is the bundle resolved from spec caBundleRef. Published here because
	// ConfigMaps do not replicate between shards; a CA bundle is public, so this is
	// safe to expose. Mirrors how status.identityHash publishes a non-secret value
	// derived from a Secret only the export's own shard can read.
	//
	// +optional
	CABundle []byte `json:"caBundle,omitempty"`
}

type ConfigMapKeyReference struct {
	// +required
	Namespace string `json:"namespace"`
	// +required
	Name string `json:"name"`
	// key defaults to "ca.crt".
	//
	// +optional
	// +kubebuilder:default="ca.crt"
	Key string `json:"key,omitempty"`
}
```

Note the deliberate departure from `apiextensionsv1.WebhookClientConfig`. Reusing it would
have been cheaper, but its `service` reference has no meaning in a logical cluster, since a
`Service` in a workspace has no backing endpoints and a workspace has no pod network. Its
inline `caBundle` is likewise a field providers would have to write by hand rather than
have injected. Only the `url` form is meaningful in kcp, so the type is defined explicitly
rather than inherited with most of it unusable.

`EphemeralReview` is a new non-persisted type in the same group, following
`AdmissionReview`'s layout.

## Implementation Notes (kcp repo)

1. `sdk/apis/apis/v1alpha1`: new types above, generated deepcopy/clients, and validation
   for the ephemeral/conversion and ephemeral/storage-version constraints.
2. New `pkg/ephemeral` package: webhook client (`EphemeralReview` marshalling, timeout,
   single attempt, response validation against the schema) plus a transport cache keyed by
   APIExport + group/resource. Server trust comes from the replicated
   `status.ephemeralEndpoints[].caBundle`; the client certificate is the shard's own,
   loaded from disk via `k8s.io/apiserver/pkg/server/dynamiccertificates` so rotation
   takes effect without a restart. Note that no part of this path reads a Secret from
   another shard.
3. New REST storage implementing `rest.Creater`, `rest.Scoper`, `rest.SingularNameProvider`
   and a `rest.Storage` that advertises only `create` in discovery. Deliberately does not
   implement `rest.Lister` or `rest.Watcher`.
4. Wire the storage into `RestProviderFunc` on both the binding path and the apiexport
   virtual workspace path.
5. `pkg/permissionclaim` and `pkg/reconciler/apis/permissionclaimlabel`: treat
   schema-declared ephemeral resources as non-persisted.
6. APIExport reconciler (runs on the export's own shard): validate that ephemeral schemas
   and `ephemeralEndpoints` line up, resolve `caBundleRef` from the provider workspace,
   publish the bytes to `status.ephemeralEndpoints[].caBundle`, and surface the result as
   an `EphemeralEndpointsValid` condition. Endpoint misconfiguration should be visible on
   the APIExport, not discovered by a consumer getting a `503`.
7. Shard client identity: mint a client certificate per shard from a kcp-owned CA distinct
   from the front-proxy requestheader CA, and publish that CA bundle where providers can
   fetch it.
8. e2e: an ephemeral schema in an APIExport, a test webhook, and assertions that (a) the
   response body matches what the webhook returned, (b) `list` is empty, (c) nothing
   appears in etcd under the export's identity prefix, (d) `failurePolicy: Fail`
   surfaces `503`, (e) a webhook that does not require a client certificate is still
   called with one, and (f) the resource is served correctly when the consumer's workspace
   and the APIExport are on **different shards**, the case that rules out any design
   depending on cross-shard Secret access.

### Suggested POC scope

To answer the "let's see it in action" bar before committing to the API: a branch that
hardcodes one ephemeral resource served through the apiexport virtual workspace with a
fixed webhook URL and a fixed client certificate on disk, no new CRD fields, no
permission-claim work. That is enough to
demonstrate the request path end to end and to expose whatever is wrong with the response
contract before it is frozen in an API.

## Alternatives Considered

**Serve ephemeral resources only through virtual workspaces.** Suggested in discussion and
appealing because the VW already terminates requests outside the storage path. It solves
the provider-side case but not the consumer-side one: a client that bound the APIExport
expects the resource in its own workspace, not at a separate VW URL. Since the same REST
storage serves both, restricting to VWs buys nothing and costs the primary use case.

**Provider-supplied client certificates.** The provider generates a keypair, references it
from the APIExport, and kcp presents it. Rejected on the cross-shard constraint in section
1a: the Secret is readable only on the export's own shard, and every way around that is
worse than the problem: replicating private keys into every shard's etcd, proxying
webhook calls through the export's shard, or making providers issue one certificate per
shard and track kcp's topology. The last of these also shows the granularity is wrong:
a per-shard certificate identifies kcp, not the provider relationship, so per-provider
issuance grants no authorization capability that a kcp-published CA does not.

**Put the whole webhook configuration on the APIResourceSchema, next to `conversion`.**
This was the first shape of this proposal and it is wrong for one decisive reason: the
schema spec is immutable. A CA bundle or endpoint URL frozen at schema creation cannot be
rotated, so a routine certificate renewal would force a new `APIResourceSchema` and an
`APIExport` update, a schema-change event visible to every consumer, caused by an
operational detail they should never see. Immutability is right for *what the API is* and
wrong for *where it is served*, hence the split.

**A single `ephemeralwebhook` endpoint per APIExport, multiplexing all its resources.**
Simpler still, but it forces one backend to serve every ephemeral resource in the export
and gives no way to point two resources at two different services. The per-resource list
costs little and keeps that option open.

**Also serve `get`.** A named `get` reads more naturally than `POST`ing an object, and it
is the first thing reviewers will ask for. It is rejected rather than deferred, on the
encoding grounds in section 2a: there is no safe way to put an arbitrary `spec` in a URL,
and the one mechanism Kubernetes has for it drops nested and repeated fields silently.
Secondary objections, `get` implying `list` and inviting clients to treat the answer as
cacheable, point the same way.

**Extend workspace mounts to per-resource granularity.** Raised in discussion as "sounds
more like something for a mounted service". `Workspace.spec.mount` already delegates a
workspace to an external URL, so the routing machinery partly exists. Narrowing it from a
workspace to a single resource path is a much larger change than it appears: mounts hand
off the whole request, including discovery, authentication and authorization, whereas an
ephemeral resource must stay inside kcp's discovery, RBAC and schema validation and only
delegate the *answer*. Different unit, different trust boundary, different failure mode.
The webhook contract keeps kcp in control of everything except the response body.

**Reuse `CustomResourceConversion`-style plumbing without new types.** Conversion webhooks
have no `userInfo` and no way to return an API error, both of which are load-bearing here.

## Risks and Mitigations

- **Control plane availability now depends on provider webhooks.** A slow webhook holds a
  kcp request goroutine for up to `timeoutSeconds`. Mitigated by a hard 30s cap, no
  retries, and the fact that the blast radius is confined to the ephemeral resource
  itself, so no persisted resource's availability is affected. This is the same exposure
  admission webhooks already create, with a narrower reach.
- **Amplification.** A client can drive webhook load at whatever rate the API server
  accepts requests. Standard API priority and fairness applies; providers should treat the
  webhook as an internet-facing service.
- **Response size.** A webhook could return a very large object. kcp should enforce the
  same request-body limit on the response as on the incoming request.
- **Client surprise.** `kubectl create -f bucketinfo.yaml -o yaml` works and prints the
  answer; `kubectl get bucketinfos` returns nothing. This mirrors
  `SubjectAccessReview` exactly, but it is unfamiliar to users who have not met that
  pattern. Discovery should advertise only `create`, and documentation should lead with
  the analogy.
- **The webhook is trusted to act on asserted identity.** kcp tells the endpoint who the
  user is; a compromised or spoofed caller could ask for anyone's data. Required client
  certificates are the mitigation, and the reason they are required rather than optional.
  Providers should verify the certificate chain, not merely require *a* certificate.
- **One kcp identity across all providers.** Because kcp presents the same identity to
  every ephemeral endpoint, a provider that obtains kcp's client key can impersonate kcp
  to *every other* provider's endpoint. This is the same exposure the aggregation layer
  carries with `--proxy-client-cert-file`, and the same mitigation applies: the key stays
  on the shard, never in an API object, and never in a provider-controlled workspace.
- **`caBundleRef` must not become a cross-workspace read primitive.** kcp resolves it only
  within the APIExport's own workspace and publishes only the bundle, which is public. A
  reference pointing elsewhere is rejected rather than followed.
- **Providers reaching for it as a general RPC mechanism.** A create-only, non-listable,
  webhook-backed resource is a tempting way to smuggle arbitrary RPC into the API surface.
  The schema requirement is the main guard: the response must validate against a declared
  OpenAPI schema.

## Open Questions

1. **How does kcp publish its client CA to providers?** The identity itself is settled,
   since kcp mints it per shard, but providers need the CA bundle to verify against, and
   kcp
   has no channel for that today. Candidates: a field on `APIExport.status` written by the
   same reconciler that resolves `caBundleRef`; a well-known ConfigMap materialized into
   every workspace, mirroring `extension-apiserver-authentication`; or simply a documented
   endpoint on the front-proxy. The status field is the least machinery and reuses a path
   this KEP already needs.

2. **Sharding and endpoint selection.** Stateless serving means any shard works, but if
   providers want shard-local webhook endpoints, some templating or `EndpointSlice`-like
   selection is needed. Deferred until someone wants it.
