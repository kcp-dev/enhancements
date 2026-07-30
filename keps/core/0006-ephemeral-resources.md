# Ephemeral Resources Enhancement Proposal

## Summary

A resource exposed through an `APIResourceSchema` and served as a CRD is backed by etcd: a
client `POST` writes an object, a controller reconciles it, and the client polls or
watches for `status`. For a whole class of provider APIs this is the wrong shape. The
client is asking a *question* and wants an *answer*, not a record.

Kubernetes already has this shape internally. `SubjectAccessReview`, `SelfSubjectReview`
and `TokenReview` are submitted with `POST`, answered synchronously in the response body,
and never persisted. They are implemented as bespoke REST storage inside the API server,
so the pattern is unavailable to anyone extending the API surface.

This proposal introduces **ephemeral resources**: an `APIExport` may declare that a
resource it exposes is served ephemerally, nominating a webhook that answers requests
against it. kcp serves the resource as a normal, discoverable, RBAC-governed API; on
`create` it forwards the submitted object to the provider's webhook and returns the
webhook's response object to the client. Nothing reaches storage.

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
fit kcp's model. What is missing is a way for a *provider* to declare one, and a place to
declare it: `ResourceSchemaStorage` in v1alpha2 already enumerates how a resource is
served, with `crd` and `virtual` variants.

### Goals

1. Allow an `APIExport` to declare a resource it exposes as ephemeral: served,
   discoverable, RBAC-governed, never written to etcd.
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
5. **No change to how persisted resources behave.** A resource whose storage is `crd` or
   `virtual` is unaffected.

## Proposal

### 1. Declaring an ephemeral resource

kcp v1alpha2 already models how a resource is served. `APIExportSpec.Resources[]` carries a
`Storage` field with exactly one variant set:

```go
// +kubebuilder:validation:XValidation:rule="has(self.crd) != has(self.virtual)",...
type ResourceSchemaStorage struct {
	CRD     *ResourceSchemaStorageCRD     `json:"crd,omitempty"`
	Virtual *ResourceSchemaStorageVirtual `json:"virtual,omitempty"`
}
```

Ephemeral serving is a third storage kind, so it goes here rather than into a new parallel
structure. The `APIResourceSchema` needs no marker and no new field at all: it stays an
ordinary schema, and the APIExport decides how instances of it are served. The same schema
could be served as a CRD by one export and ephemerally by another.

```yaml
apiVersion: apis.kcp.io/v1alpha2
kind: APIExport
metadata:
  name: s3.example.com
spec:
  resources:
  - name: bucketinfos
    group: s3.example.com
    schema: v1alpha1.bucketinfos.s3.example.com
    storage:
      ephemeral:
        url: https://s3-info.example.com/ephemeral/bucketinfos
        caBundle: <base64 PEM>
        timeoutSeconds: 10
        failurePolicy: Fail
```

This placement carries three properties the resource needs, none of which required new
machinery:

- **Mutable.** `APIResourceSchema.spec` is immutable in its entirety, since
  `ValidateAPIResourceSchemaUpdate` rejects any spec change with `"is immutable"`. A URL or
  a CA bundle frozen there could not be rotated without minting a new schema and
  repointing the APIExport, which is a schema-change event visible to every consumer,
  caused by an operational detail they should never see. The APIExport is mutable, so
  rotation is an ordinary update.
- **Replicated.** The APIExport is on the cache server's replication list, so every shard
  sees the URL and CA bundle without any cross-shard lookup. This is why `caBundle` is
  inline rather than a ConfigMap reference: a reference would have to resolve in the
  provider's workspace, which only the provider's own shard can read.
- **Already keyed by resource.** No second list, and no validation rule to keep two lists
  in agreement.

Validation rules:

- `XValidation` on `ResourceSchemaStorage` becomes exactly-one-of `crd`, `virtual`,
  `ephemeral`.
- `url` must be `https`. There is no `service` variant, since a `Service` in a logical
  cluster has no backing endpoints.

Two things a schema version may declare are simply not exercised on this path, and neither
is grounds for rejecting the export. `subresources` (`status` and `scale`, the same pair
vanilla CRDs allow) are never addressable, because only `create` is served and there is no
named object to hang `/status` off; the `status` of an ephemeral object is part of the
single response, the same way `SelfSubjectReview` returns its `status` inline.
`conversion` is never invoked, because there is no stored version to convert from: the
requested version is passed to the webhook, which answers in it. Both remain meaningful
when the same schema is served as a CRD by another APIExport.

There is deliberately no field for a client certificate.

### 1a. Why kcp issues its own client certificate

Client authentication is not optional here. The aggregation layer mandates mutual TLS.
The kube-apiserver presents `--proxy-client-cert-file` and the extension server verifies
it against the requestheader CA, and it does so for a specific reason: the aggregated
server is being *told* who the user is and must know the assertion came from the API
server. `EphemeralReview.request.userInfo` puts an ephemeral webhook in exactly that
position. Without client authentication, anyone who can reach the endpoint asserts
arbitrary identity and reads any user's data.

A provider-supplied certificate cannot deliver that. It would have to be referenced from a
Secret in the provider's workspace, and kcp does not replicate Secrets between shards.
`pkg/reconciler/cache/replication/replication_controller.go` carries public API metadata
only; `secrets` appears nowhere in it. An ephemeral request arrives at the shard hosting
the *consumer's* workspace, which generally is not the shard hosting the provider's, so
that shard could not read the credential. Every way around this is worse than the problem:
replicating the Secret would copy private keys into every shard's etcd, proxying the call
through the provider's shard would reintroduce shard affinity for a request with no stored
state (see section 3), and issuing one certificate per shard would make providers track
kcp's topology.

That last option also exposes the deeper point. Once the certificate is per-shard, it
identifies *kcp*, not the provider relationship, so per-provider or per-binding
granularity buys nothing. There is no authorization decision a webhook can make from
"this is the cert I issued to kcp" that it cannot make from "this is kcp, verified against
kcp's published CA".

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
lists, and the labeler additionally consults the binding's APIExport: a resource whose
storage is `ephemeral` is non-persisted and non-claimable. The check in
`permissionclaim_labeler.go` and the skip in
`pkg/reconciler/apis/permissionclaimlabel/permissionclaimlabel_reconcile.go` both need to
account for this.

Whether ephemeral resources should later become *claimable*, so an export can claim
another export's ephemeral resource and call through it, is deliberately left open. It
is additive and should follow real demand.

## API Changes

All changes are in v1alpha2 `types_apiexport.go`
(`staging/src/github.com/kcp-dev/sdk/apis/apis/v1alpha2/`).

```go
// ResourceSchemaStorage defines how the resource is stored.
//
// +kubebuilder:validation:XValidation:rule="[has(self.crd), has(self.virtual), has(self.ephemeral)].exists_one(x, x)",message="Exactly one of crd, virtual or ephemeral must be set"
type ResourceSchemaStorage struct {
	// ... existing crd and virtual fields ...

	// Ephemeral storage defines that instances of this resource are never
	// persisted. Each create is answered synchronously by the configured webhook
	// and nothing is written to etcd. Only the create verb is served.
	//
	// +optional
	Ephemeral *ResourceSchemaStorageEphemeral `json:"ephemeral,omitempty"`
}

// ResourceSchemaStorageEphemeral describes the webhook that answers requests for
// an ephemeral resource.
type ResourceSchemaStorageEphemeral struct {
	// URL is the HTTPS endpoint kcp calls. Plain http is rejected. There is no
	// service variant: a Service in a logical cluster has no backing endpoints.
	//
	// +required
	// +kubebuilder:validation:Pattern=`^https://`
	URL string `json:"url"`

	// CABundle is a PEM-encoded CA bundle used to verify the endpoint's serving
	// certificate. Inline rather than a reference because the APIExport replicates
	// between shards while Secrets and ConfigMaps do not, and because the APIExport
	// is mutable, so rotating it is an ordinary update. Defaults to the system trust
	// store when empty.
	//
	// +optional
	CABundle []byte `json:"caBundle,omitempty"`

	// TimeoutSeconds bounds how long kcp waits for a response. Defaults to 10.
	//
	// +optional
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=30
	// +kubebuilder:default=10
	TimeoutSeconds int32 `json:"timeoutSeconds,omitempty"`

	// FailurePolicy determines the behaviour when the webhook cannot be reached.
	// Fail returns 503 to the client; Ignore returns the submitted object unchanged.
	//
	// +optional
	// +kubebuilder:validation:Enum=Fail;Ignore
	// +kubebuilder:default=Fail
	FailurePolicy string `json:"failurePolicy,omitempty"`
}
```

There is intentionally no client certificate field. kcp presents its own identity and
publishes the CA for providers to verify against, for the reasons in section 1a.

`EphemeralReview` is a new non-persisted type in the same group, following
`AdmissionReview`'s layout.

Nothing is added to `APIExportStatus`. An earlier draft resolved a CA bundle reference
into status so that other shards could see it; putting the bundle inline in the spec makes
that unnecessary, since the spec replicates already.

## Implementation Notes (kcp repo)

1. `sdk/apis/apis/v1alpha2`: the `Ephemeral` variant and its type, generated
   deepcopy/clients, and the three-way `XValidation` rule. The v1alpha1 conversion path
   changes only to refuse round-tripping an ephemeral resource into v1alpha1, which has no
   way to express it.
2. New `pkg/ephemeral` package: webhook client (`EphemeralReview` marshalling, timeout,
   single attempt, response validation against the schema) plus a transport cache keyed by
   APIExport + group/resource. Server trust comes from the inline `caBundle` on the
   APIExport spec, which every shard already has via replication; the client certificate is
   the shard's own, loaded from disk via `k8s.io/apiserver/pkg/server/dynamiccertificates`
   so rotation takes effect without a restart. No part of this path reads a Secret or
   ConfigMap from another shard.
3. New REST storage implementing `rest.Creater`, `rest.Scoper`, `rest.SingularNameProvider`
   and a `rest.Storage` that advertises only `create` in discovery. Deliberately does not
   implement `rest.Lister` or `rest.Watcher`.
4. Wire the storage into `RestProviderFunc` on both the binding path and the apiexport
   virtual workspace path, selected on the resource's storage variant.
5. `pkg/permissionclaim` and `pkg/reconciler/apis/permissionclaimlabel`: treat resources
   whose APIExport storage is `ephemeral` as non-persisted.
6. APIExport reconciler: surface endpoint validity as an `EphemeralEndpointsValid`
   condition, so a malformed URL or unparseable CA bundle is visible on the APIExport
   rather than discovered by a consumer getting a `503`.
7. Shard client identity: mint a client certificate per shard from a kcp-owned CA distinct
   from the front-proxy requestheader CA, and publish that CA bundle where providers can
   fetch it.
8. e2e: an APIExport with an ephemeral resource, a test webhook, and assertions that (a)
   the response body matches what the webhook returned, (b) `list` is empty, (c) nothing
   appears in etcd under the export's identity prefix, (d) an unreachable webhook surfaces
   `503`, (e) a webhook that does not require a client certificate is still called with
   one, and (f) the resource is served correctly when the consumer's workspace and the
   APIExport are on **different shards**, the case that rules out any design depending on
   cross-shard Secret access.

### Suggested POC scope

To answer the "let's see it in action" bar before committing to the API: a branch that
hardcodes one ephemeral resource served through the apiexport virtual workspace with a
fixed webhook URL and a fixed client certificate on disk, no new API fields, no
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
from the APIExport, and kcp presents it. Rejected on the replication constraint in section
1a: the Secret is readable only on the export's own shard, and every way around that is
worse than the problem: replicating private keys into every shard's etcd, proxying
webhook calls through the export's shard, or making providers issue one certificate per
shard and track kcp's topology. The last of these also shows the granularity is wrong:
a per-shard certificate identifies kcp, not the provider relationship, so per-provider
issuance grants no authorization capability that a kcp-published CA does not.

**Put the webhook configuration on the APIResourceSchema, next to `conversion`.** This was
the first shape of this proposal and it is wrong for one decisive reason: the schema spec
is immutable. A CA bundle or endpoint URL frozen at schema creation cannot be rotated, so
a routine certificate renewal would force a new `APIResourceSchema` and an `APIExport`
update, a schema-change event visible to every consumer, caused by an operational detail
they should never see. Immutability is right for *what the API is* and wrong for *how it
is served*.

**A marker on the schema plus a separate endpoint list on the APIExport.** The second
shape: `spec.ephemeral: {}` on the schema declaring the property, and
`spec.ephemeralEndpoints[]` on the APIExport carrying the configuration. It preserves
mutability but duplicates the resource key in two places and needs a validation rule to
keep them in agreement. `ResourceSchemaStorage` already expresses "how is this resource
served" on the APIExport, keyed by the resource entry, so the marker turns out to be
redundant: the storage variant *is* the declaration.

**A single ephemeral webhook per APIExport, multiplexing all its resources.** Simpler
still, but it forces one backend to serve every ephemeral resource in the export and gives
no way to point two resources at two different services. Per-resource configuration falls
out of `ResourceSchemaStorage` for free.

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
- **The CA bundle is world-readable.** It sits inline in the APIExport spec, which
  replicates to every shard and is visible to anyone who can read the export. A CA bundle
  is public by nature, so this is not a leak, but it does mean the field must never be
  repurposed to carry anything secret.
- **Providers reaching for it as a general RPC mechanism.** A create-only, non-listable,
  webhook-backed resource is a tempting way to smuggle arbitrary RPC into the API surface.
  The schema requirement is the main guard: the response must validate against a declared
  OpenAPI schema.

## Open Questions

1. **How does kcp publish its client CA to providers?** The identity itself is settled,
   since kcp mints it per shard, but providers need the CA bundle to verify against, and
   kcp has no channel for that today. Candidates: a field on `APIExport.status` written by
   the reconciler that validates the ephemeral endpoints; a well-known ConfigMap
   materialized into every workspace, mirroring `extension-apiserver-authentication`; or 
   simply a documented endpoint on the front-proxy. The status field is the least 
   machinery and reuses a path this KEP already needs.

2. **Sharding and endpoint selection.** Stateless serving means any shard works, but if
   providers want shard-local webhook endpoints, some templating or `EndpointSlice`-like
   selection is needed. Deferred until someone wants it.
