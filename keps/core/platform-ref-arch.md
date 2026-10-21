# Platform Reference Architecture

## Summary

Define a reference architecture for building multi-tenant platforms on kcp where application teams get workspace-based access to Kubernetes ecosystem tools (ArgoCD, Flux, Crossplane, KRO, Argo Workflows) without direct cluster access. Resources are synced to physical clusters via api-syncagent and status flows back transparently.

## Motivation

kcp provides workspace isolation, APIExport/APIBinding, and virtual workspaces — but there is no canonical guidance on how to compose these primitives into a production platform. Platform teams are left to figure out the integration patterns themselves.

This reference architecture demonstrates that kcp + api-syncagent can serve as the control plane for a multi-tenant platform that exposes Kubernetes-native tools as managed services, with strong tenant isolation and no direct cluster access.

### Goals

1. Provide a complete, deployable reference architecture with manifests and documentation.
2. Define workspace hierarchy patterns: provider workspaces (APIExports), tenant workspaces (APIBindings), WorkspaceTypes with auto-provisioning.
3. Show per-tool integration patterns for ArgoCD, Flux, Crossplane, KRO, and Argo Workflows using api-syncagent PublishedResources.
4. Identify and document gaps in kcp core and api-syncagent that block or degrade the platform experience.

### Non-Goals

* Replacing existing platform products — this is a reference, not a product.
* Defining a scheduler for workspace-to-cluster placement (covered separately).
* Implementing the gap fixes themselves — those are tracked as separate enhancements.

## Proposal

### Architecture Overview

```
root
 +-- platform-providers              (universal)
 |    +-- argocd-provider            APIExport: argocd.argoproj.io
 |    +-- flux-provider              APIExport: flux.toolkit.fluxcd.io
 |    +-- crossplane-provider        APIExport: crossplane.io
 |    +-- kro-provider               APIExport: kro.run
 |    +-- argo-workflows-provider    APIExport: workflows.argoproj.io
 |
 +-- tenants                         (organization)
      +-- team-alpha                 (application WorkspaceType)
      |    APIBindings: argocd, flux
      +-- team-beta
           APIBindings: crossplane, kro, argo-workflows

              Physical Clusters (never exposed to users)
              ==========================================
 cluster-a:  ArgoCD + Flux + api-syncagent
 cluster-b:  Crossplane + KRO + api-syncagent
 cluster-c:  Argo Workflows + api-syncagent
```

### Data Flow

1. Platform team creates provider workspaces with APIExports and APIResourceSchemas derived from tool CRDs.
2. Platform team deploys api-syncagent on physical clusters with PublishedResource configs pointing at the APIExport virtual workspace.
3. Application teams get `application`-type workspaces with auto-provisioned APIBindings via WorkspaceType `defaultAPIBindings`.
4. Users create resources (e.g., ArgoCD Application) in their kcp workspace.
5. api-syncagent syncs resources to the physical cluster where the operator reconciles them.
6. Status flows back through api-syncagent to the user's kcp workspace.

### Design Decisions

| Decision | Rationale |
|----------|-----------|
| Per-workspace namespace isolation on physical clusters | Tenant isolation using K8s RBAC, NetworkPolicy, ResourceQuota |
| One APIExport per tool | Simplifies consumer binding |
| Provider workspaces under `root:platform-providers` | Centralized management, predictable paths |
| `application` WorkspaceType with `defaultAPIBindings` + `Maintain` lifecycle | Auto-provision on creation, day-2 updates propagate |

### Security Model

- Workspace-scoped RBAC — no cross-tenant visibility.
- APIExport permission claims with explicit consumer acceptance.
- Per-workspace namespaces on physical clusters with K8s RBAC and NetworkPolicy.
- Maximal permission policy on APIExports to bound consumer permissions.

## Known Gaps

These gaps are identified as part of this work and should be addressed as separate enhancements or api-syncagent improvements:

| # | Gap | Component | Priority |
|---|-----|-----------|----------|
| 1 | Cross-CRD reference sync (related resources beyond ConfigMap/Secret) | api-syncagent | CRITICAL |
| 2 | Subresource proxying (Pod logs/exec through kcp) | kcp core + api-syncagent | CRITICAL |
| 3 | Event forwarding from physical clusters | api-syncagent | IMPORTANT |
| 4 | Dynamic CRD-to-APIResourceSchema automation (for Crossplane/KRO) | New controller | IMPORTANT |
| 5 | Admission/validation forwarding (dry-run proxy) | api-syncagent | IMPORTANT |
| 6 | Status aggregation/normalization templates | Contrib | IMPORTANT |
| 7 | Multi-cluster scheduling via Partition/PartitionSet | kcp core | NICE-TO-HAVE |
| 8 | Per-workspace resource quota on physical clusters | kcp core | NICE-TO-HAVE |
| 9 | Self-service API catalog for optional bindings | New controller | NICE-TO-HAVE |

## Deliverables

1. **Manifests** — WorkspaceTypes, APIResourceSchemas, APIExports, PublishedResources for all 5 tools.
2. **Documentation** — Architecture overview, per-tool integration guides, gap analysis.
3. **Examples** — End-to-end usage examples for each tool.
4. **Location** — `contrib/platform-ref-arch/` in the kcp repository.

## Alternatives Considered

* **Per-tool separate proposals** — rejected because the value is in the composition pattern, not individual integrations.
* **Ship as a separate repository** — rejected because tight coupling with kcp core and api-syncagent makes co-location beneficial for testing and iteration.
