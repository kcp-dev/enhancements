# kcp hub: an extended reference architecture

- Repository: external, outside the `kcp-dev` organisation (name to be defined)
- Status: proposal, for kcp maintainer awareness and endorsement
- Owner: Mangirdas Judeikis

## Summary

kcp is an engine for building platforms. A newcomer who wants users,
organisations, a catalog of services and a way to enable one has to assemble
that from a dozen repositories and thirty concept pages, with no working
example that runs end to end. Many give up before they get there.

This proposal establishes hub, a running end-to-end reference of how to
build on kcp using upstream best practices, in its own repository outside
the `kcp-dev` organisation. It contains
tenancy, a producer catalog, a thin (emphasis on thin!) console, and the
deployment that wires it to an identity provider. It is usable as-is as a
base layer and states clearly where "your platform" starts.

The existing reference setup (multi-shard deployment via the operator) stays
where it is. Hub is the layer above it, deployable independently.

## Motivation

### What an adopter has to learn today

> A note for maintainers: the list below looks obvious to us because we have
> lived with it for years. Read it as someone seeing kcp for the first time.

Counted against the current `kcp-dev/kcp` tree and sibling repositories:

| Area | Count |
|---|---|
| Concept documentation pages | 30 |
| kcp-specific API kinds, across 6 API groups | 16 |
| Repositories needed for a real deployment with controllers | 8 or more |

The kinds are `Workspace`, `WorkspaceType`, `WorkspaceAuthenticationConfiguration`,
`LogicalCluster`, `Shard`, `APIExport`, `APIBinding`, `APIResourceSchema`,
`APIExportEndpointSlice`, `APIConversion`, `CachedObject`, `ClusterCachedResource`,
`ClusterCachedResourceEndpointSlice`, `Partition`, `PartitionSet`, and
`LogicalClusterMigration`. The repositories are `kcp`, `kcp-operator`, `helm-charts`,
`client-go`, `apimachinery`, `code-generator`, `multicluster-provider` and
`virtual-workspace-framework`, plus `api-syncagent` and `init-agent` for
common cases.

Then there are concepts without a kind: logical clusters and workspace paths,
the front-proxy, the authorizer chain, permission claims and maximal
permission policies, export identities, initializers, mounts, the cache
server, virtual workspaces, and cluster-aware clients and informers.

This is documented as a flat list. Which parts a platform builder touches,
and in what order to learn them, is not written down anywhere.

### What is missing to have a platform

After learning the above, a builder still has to design and write the
following without upstream guidance:

- **Identity to kubeconfig.** How an OIDC user becomes a kcp user and gets a
  kubeconfig for the workspaces they were granted, without an admin
  credential in the path.
- **Tenancy.** How organisations, projects and memberships map to workspaces,
  WorkspaceTypes and RBAC, and which of those a user may create themselves.
- **A catalog.** How a provider publishes an API, how a consumer discovers and
  enables it, and how consent for permission claims is expressed, using
  APIExport and APIBinding rather than a parallel model.
- **A producer.** A minimal controller that reconciles its kind in every
  workspace that bound it, with generated schema and a deployment.
- **A console.** Something a non-kubectl user can use.
- **A CLI.** One command for users to log in, pick a tenant, enable a
  service and get a kubeconfig, without the console or the kcp plugins.
- **A deployment.** Charts that install the above against a kcp deployment,
  and a one-command local variant for development.

**Every adopter solves these again, and often with mistakes that only show up
too late in the development cycle, often after users have already been affected.**

The mistakes we have seen cluster in the integration layers: the UI, the CLI,
and the end-to-end flow from login to a working kubeconfig. Each on its own
is simple. Getting them to agree on identity, authorization and workspace
paths is where it goes wrong, and it is what no upstream example covers.

### Why outside kcp-dev

kcp is an engine and should stay one. A reference platform takes opinions
on tenancy, catalog and UI that the engine should not take, and hosting it
under `kcp-dev` would change the project's scope and add load to maintainers
who did not sign up for it. Keeping hub in its own repository keeps those
concerns separate: kcp stays unopinionated, hub can move fast, and nothing
in kcp depends on it.

What hub asks of the kcp project is small: the standalone components it
builds on stay in `kcp-dev/contrib-virtual-workspaces`, hub is linked from
the kcp documentation as a worked example of building on kcp, and this
document records the scope so the two projects do not drift apart.

## Goals

1. A newcomer can run `make up` and, within a few minutes, log in as an OIDC user,
   create an organisation and a project, enable a service from the catalog,
   and reach it with a kubeconfig that holds only their own identity.
2. Every piece of hub follows an upstream best practice, and each README
   explains why it is shaped that way in terms of kcp's authorization model.
3. Hub is usable as-is as a base layer. A platform team can deploy it and
   build on top without forking it.
4. The tenancy and catalog models are pluggable. The interfaces are the
   contract; the shipped implementations are one example each.

## Non-goals

1. **A product.** No commercial support, no SLA, no billing, no invite flows,
   no branding, no per-customer features. Those are built on top.
2. **A framework on top of a framework.** Hub adds no abstraction over kcp's
   APIs. Enabling a service is an APIBinding. Membership is RBAC.
3. **Arbitrary provider implementations.** Hub ships a producer skeleton and
   a small fixed set of example producers. Other providers live in their own
   repositories. A provider that fulfils the producer interface works on hub
   and on any other kcp-based platform that honours the same interface, so
   there is no reason to host it in hub.
4. **Replacing the deployment reference.** Multi-shard operator deployments
   stay documented where they are. Hub consumes them.
5. **Competing with downstream platforms.** Hub stops where a platform's
   opinions start, as written in the boundary document.

## Proposal

### Scope

Hub is six pieces, each deployable alone, composed only through kcp's own
APIs. The console, the CLI and the end-to-end flow are the point; the rest is
what they need to run.

| Piece | What it is | Where the line is |
|---|---|---|
| **tenancy** | Organisations, projects and memberships as workspaces, WorkspaceTypes and RBAC. A "which tenants am I in" review. The model is an interface; this is one implementation. | No invite flows, no SSO-driven auto-provisioning, no quotas. |
| **catalog** | Producers publish by annotating their own APIExport. Consumers enable by creating an APIBinding. A read-only virtual workspace serves the catalog. | No approval workflows, no pricing, no private catalogs beyond RBAC. |
| **producer skeleton** | One copyable Go module: a kind, generated schema, a multi-workspace controller, instructions, Dockerfile, deployment. Two or three example producers of different shapes, for instance an MCP endpoint and a composition-based service. | Examples stay small. No real backing infrastructure. |
| **console** | Static files talking to the front-proxy with the user's own token. Sign in, see tenants, browse the catalog, enable, inspect a workspace. | Basic CRUD only. No backend of its own, nothing privileged. |
| **CLI** | One binary over the same surfaces as the console: login, choose tenant and project, list and enable offerings, manage memberships, write a kubeconfig. Importable as cobra commands so a platform can ship its own CLI on top. | No admin tooling beyond what the walkthrough needs. |
| **deployment** | Two paths to the same result: Helm charts that install the pieces above against an existing kcp, or kcp-operator resources (shards, front-proxy, virtual workspaces) that bring up kcp and hub together in one shot. Both wire to an OIDC issuer. Deployable to production as a bare-bones control plane. A local one-command variant (kind, Dex, seed data) uses the operator path for development and CI. | Sizing, HA and multi-region guidance stay in the kcp deployment reference. No managed-service tooling. |

### Deliverables

1. The hub repository with the six pieces, CI, images, and end-to-end tests
   that execute the walkthrough in goal 1.
2. A walkthrough from zero to an enabled service, in the order a builder
   should learn kcp, with each step naming the kcp concept it exercises.
   Linked from kcp's docs as a worked example.
3. A boundary document in the repository stating what hub will and will not
   accept, referenced from the contributing guide.

### Governance

- Hub has its own organisation, OWNERS, CI and release cadence. The initial
  approvers are the people already working on it. kcp maintainers are not
  expected to review or maintain it, and kcp governance does not apply.
- Frontend code is reviewed by approvers with frontend experience, listed in
  OWNERS for the `ui` directory.
- Hub tracks kcp's latest minor release and is tested against it.
- Hub is experimental. If it proves itself, moving it under `kcp-dev` can be
  proposed later, as a separate decision.

### Relationship to existing work

The standalone components in `kcp-dev/contrib-virtual-workspaces` (tenancy,
marketplace, access, mcp and the producer quickstart) stay there and remain
usable on their own. Hub is the CLI, the UI and the wiring that composes
them into one deployment. Prototype code for all of this exists and runs.

## Risks

- **Picking winners.** A reference implies a recommendation. Mitigation: the
  tenancy and catalog models are interfaces with one shipped implementation.
- **Alienating downstream platforms.** Mitigation: hub is a base layer and a
  teaching tool for platform builders.
- **Drift from upstream.** An external reference can fall behind kcp or
  take shortcuts kcp would not. Mitigation: CI pinned to kcp releases,
  the kcp-facing components stay upstream in contrib-virtual-workspaces, and
  contributors overlap with kcp.
- **Maintenance load on kcp.** None by construction. Nothing in kcp's CI or
  release depends on hub.
- **Frontend review.** Mitigation: named frontend approvers. The console is
  static CRUD with no privileged backend.

## Alternatives

- **Documentation only.** Write the walkthrough without the code. Text
  cannot be run, and authorization mistakes only show up when something
  executes. Still worth doing alongside hub, not instead of it.
- **Subproject under kcp-dev.** Gives the project control of the reference
  and keeps the community in one place, at the cost of a scope change for
  kcp, a formal vote, and maintainer load for code they did not choose.
  Possible later if hub proves itself.
- **Extend the deployment reference.** Add api-syncagent and examples to the
  multi-shard setup. That shows how to run kcp, not how to build on it.
  Complementary to hub and can happen independently.
