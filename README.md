## capi-work

A hands-on learning repo for **Cluster API (CAPI)** using Kubernetes to manage the
full lifecycle of other Kubernetes clusters (provision, upgrade, operate, delete)
declaratively.

The goal: go from *"10 clusters in different AWS accounts, upgraded manually"* to
*"clusters as custom resources in one management cluster, upgraded by changing a
field"*,  first locally (Docker provider), then on EKS (CAPA).

### Roadmap

- [x] **01 — Why and What CAPI?**: the problem (cluster sprawl) and what CAPI solves
- [ ] **02 — Preconditions**: tools, versions, host requirements
- [ ] **03 — Setup** management cluster + `clusterctl init` + first workload cluster
- [ ] **04 — Test**: declarative day-2 ops, i.e. a Kubernetes version upgrade
- [ ] **05 — Recommendations**: production guidance and gotchas

## 01 - Why Cluster API?

### The problem: cluster sprawl

With one or two clusters, you can manage them any way you like a bash script,
`eksctl`, Terraform, or even the console. It works.

The pain starts when you run **many clusters across environments and providers**:
dev / test / prod, often in **different cloud accounts**, sometimes across different
clouds or bare metal. Each environment tends to pull in its own toolsb or at least
its own flavor of the same tools and you drift into what the community calls
**cluster sprawl**: inconsistent provisioning, inconsistent upgrades, no single place
that knows the state of everything.

This is exactly the multi-cluster problem: provisioning, upgrading, and operating a
fleet of clusters *consistently*.

### What Cluster API is

Cluster API (CAPI) is a Kubernetes sub-project that provides a declarative API to
manage the full lifecycle of Kubernetes clusters, create, upgrade, scale, operate,
and delete.

The core idea is **using Kubernetes to manage Kubernetes**:

- A **management cluster** runs the CAPI controllers and Custom Resource Definitions
  (CRDs). It is the single place from which you manage everything.
- It provisions and manages **workload clusters** (also called *child* clusters), the
  clusters that actually run your applications.
- **Everything is a custom resource**: `Cluster`, `Machine`, `MachineDeployment`,
  a control-plane object, and infrastructure objects. You declare the *desired state*;
  controllers continuously reconcile reality toward it.


### Why it matters

- **Centralized management.** One place, one API, one way to manage clusters — the same
  way Kubernetes centralized workload management.
- **Provider abstraction.** A pluggable provider model (infrastructure / control-plane /
  bootstrap providers) hides the differences between AWS, Azure, GCP, bare metal, Docker,
  and more. The same workflow applies regardless of where the cluster runs.
- **Declarative day-2 operations.** Upgrades become "edit a field." Built-in guardrails
  prevent unsafe moves — e.g. CAPI refuses to upgrade worker machines ahead of the
  control plane (version skew). Health and self-healing come via `MachineHealthCheck`
  (offline nodes are replaced, like pods). Standardization comes via `ClusterClass`
  (golden templates you instantiate per environment).
- **Immutability as a safety property.** CAPI treats a machine like an immutable pod:
  you don't mutate it in place, you replace it from a trusted image. That eliminates
  configuration drift and makes troubleshooting predictable. (A recent opt-in feature
  adds *in-place updates* for low-disruption changes such as rotating an SSH key, while
  keeping rolling replacement as the default for anything that needs a drain.)

### When *not* to reach for CAPI

CAPI earns its complexity at **scale, across multiple providers, or when you need a
fleet/platform layer**. For one or two clusters on a single provider, Terraform or
`eksctl` is often simpler and perfectly fine. Don't adopt CAPI just to have it — adopt
it when sprawl, inconsistency, or manual day-2 ops are the thing that actually hurts.

### References

- The Cluster API Book — https://cluster-api.sigs.k8s.io
- [Introduction To Cluster API - Jussi Nummelin, Mirantis Inc](https://www.youtube.com/watch?v=UMNCKkdrfUo)
- [KubeCon 2026 Europe: In-place Updates with Cluster API: Fabrizio Pandini & Stefan Büringer (immutability, rollout strategies)](https://www.youtube.com/watch?v=CMf6rOPo9Z0)
