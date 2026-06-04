## capi-work

A hands-on learning repo for **Cluster API (CAPI)** using Kubernetes to manage the
full lifecycle of other Kubernetes clusters (provision, upgrade, operate, delete)
declaratively.

The goal: go from *"10 clusters in different AWS accounts, upgraded manually"* to
*"clusters as custom resources in one management cluster, upgraded by changing a
field"*,  first locally (Docker provider), then on EKS (CAPA).

### Roadmap

- [x] [01 — Why and What CAPI?](#01---why-cluster-api)
- [x] [02 — Preconditions](#02--preconditions)
- [x] [03 — Setup](#03--setup)
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

## 02 — Preconditions

Before any `clusterctl` command runs, three things must exist. Everything else on this
page is just *how* to satisfy them for the **local / Docker** path we use first.

### The three things CAPI always needs

1. **A management cluster** — an existing Kubernetes cluster that gets turned *into* the
   management cluster when we install the CAPI providers into it. Locally this is a
   `kind` cluster. It must be **at least Kubernetes v1.20.0**. (In production you'd use a
   real, backed-up cluster kept separate from any workload.)
2. **`clusterctl`** — the CAPI command-line tool. It fetches and installs provider
   components, encodes best practices, and handles day-2 operations such as provider
   upgrades.
3. **An infrastructure provider** — decides *where* workload clusters actually run.
   Locally: **Docker (CAPD)**, which runs cluster nodes as containers. On AWS: **CAPA**.
   The provider is the only piece that changes between local and cloud.

### Tooling to install (local / Docker path)

| Tool | Why you need it | Version floor |
|------|-----------------|---------------|
| **Docker** | Required. CAPD runs nodes as containers; `kind` runs on it too. | current |
| **kind** | Creates the local management (and bootstrap) cluster. | **≥ v0.31.0** |
| **kubectl** | Talks to both the management and the workload clusters. | recent |
| **Helm** | Installs some providers and CNI charts. | recent |
| **clusterctl** | The CAPI CLI — init, generate, describe, upgrade. | **v1.13.2** (current) |

Install (macOS with Homebrew):

```
brew install kind kubectl helm clusterctl
```

`clusterctl` via curl instead (macOS- AMD64):

```
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/download/v1.13.2/clusterctl-darwin-amd64 -o clusterctl
chmod +x ./clusterctl
sudo mv ./clusterctl /usr/local/bin/clusterctl
```

### Host requirements (the gotchas)

These are where local CAPD setups usually fail, so handle them up front.

- **macOS (Docker Desktop): give Docker enough memory.** The Book recommends **at least
  6 GB for a single CAPD workload cluster**. We will run **dev + test + prod**, so plan
  for roughly **10–12 GB** and reduce machine counts (1 control-plane + 1 worker each).
  If memory is tight, build one workload cluster at a time.

- **Disk:** each Kubernetes version pulls its own `kindest/node` image; several versions
  during upgrade testing add up.

### Docker-provider-specific preconditions

Two settings the Docker path *requires* — both are easy to forget and both cause
silent failures:

1. **Mount the host Docker socket into the management cluster.** CAPD creates the
   workload-cluster node containers by talking to the host's Docker, so the `kind`
   management cluster must be created with the socket mounted:
   ```yaml
   # excerpt of the kind config used in Step 3
   nodes:
   - role: control-plane
     extraMounts:
       - hostPath: /var/run/docker.sock
         containerPath: /var/run/docker.sock
   ```
   Skip this and workload clusters never leave `Provisioning`.

2. **Enable the ClusterClass feature gate** before `clusterctl init`:
   ```bash
   export CLUSTER_TOPOLOGY=true
   ```
   The Docker quickstart templates are ClusterClass-based (so config can adapt to the
   Kubernetes version). This is also *why*, in Step 4, the upgrade is driven by editing
   `spec.topology.version` on the `Cluster` object.


### Verification checklist

Run this before Step 3; every command should succeed and meet the version floors above.

```
docker info>/dev/null && echo "docker: ok"
kind version                       # expect >= v0.31.0
kubectl version --client
helm version
clusterctl version                 # expect v1.13.2
```

When all five pass, you're ready to build the management cluster in **Step 3 — Setup**.

## 03 — Setup

Goal of this step: stand up the **management cluster**, install CAPI + the **Docker
provider**, then have CAPI create the workload clusters. We build `dev` in full detail,
then repeat for `test` and `prod` so Step 4 has a small fleet to upgrade.

```
        mgmt (kind)  ← CAPI controllers + Docker provider (CAPD)
            │  creates & manages
   ┌────────┼────────┐
   ▼        ▼        ▼
  dev      test     prod      ← workload clusters (CAPD containers), all as CRDs in mgmt
```

We start every workload cluster at **v1.34.0** so Step 4 can upgrade them to **v1.35.0**
(one minor version — the same rule EKS enforces).

> Run all commands from the repo root. `*.kubeconfig` files are git-ignored (they hold
> cluster certs/endpoints) — keep them out of version control.

### 1. Create the management cluster

The kind config (`mgmt-kind.yaml`) already includes the `docker.sock` mount
CAPD needs.

```
kind create cluster --config mgmt-kind.yaml
kubectl cluster-info --context kind-mgmt
```

`kubectl` is now pointed at `kind-mgmt`. Everything in the next steps runs against this
management cluster unless a `--kubeconfig` flag says otherwise.

### 2. Install CAPI + the Docker provider

Enable the ClusterClass feature gate **before** init (this is what makes upgrades a
field edit later), then initialize:

```
export CLUSTER_TOPOLOGY=true
clusterctl init --infrastructure docker
```

`clusterctl init` installs four sets of components: the core `cluster-api` provider, the
`kubeadm` bootstrap and control-plane providers, and the `docker` infrastructure
provider. Wait for them to be running:

```
kubectl get pods -A | grep -E 'capi|capd|cluster-api'
```

Output will be similar like below:

```
kubectl get pods -A | grep -E 'capi|capd|cluster-api'

capd-system                         capd-controller-manager-75b4c5656f-4kcd4                         1/1     Running   0          3m43s
capi-kubeadm-bootstrap-system       capi-kubeadm-bootstrap-controller-manager-79f8b974d4-lg4qv       1/1     Running   0          3m44s
capi-kubeadm-control-plane-system   capi-kubeadm-control-plane-controller-manager-866c58f958-99vpt   1/1     Running   0          3m43s
capi-system                         capi-controller-manager-6c7db68fd-fcjlc                          1/1     Running   0          3m44s
```

### 3. Create the first workload cluster: `dev`

Generate the manifests into the root folder and apply them. Keep machine counts at 1+1 to
stay within the memory budget:

```
clusterctl generate cluster dev --flavor development \
  --kubernetes-version v1.34.0 \
  --control-plane-machine-count=1 \
  --worker-machine-count=1 \
  > dev.yaml

kubectl apply -f dev.yaml
```

> The `development` flavor includes a shared ClusterClass named `quick-start`; `test` and
> `prod` reuse it. Re-applying it is harmless — you just end up with three `Cluster`
> objects pointing at one ClusterClass.

Watch it provision (control plane first, then the worker):

```
kubectl get cluster
clusterctl describe cluster dev
```

### 4. Get the `dev` kubeconfig

On **macOS + Docker Desktop**, retrieve the kubeconfig with `kind`, not `clusterctl` —
the `clusterctl` method points at a container IP that the Docker Desktop VM doesn't
expose to the host:

```
kind get kubeconfig --name dev > dev.kubeconfig

sed -i '' 's#https://0.0.0.0:#https://127.0.0.1:#' dev.kubeconfig
# kind get kubeconfig on macOS/Docker Desktop sometimes writes the server as https://0.0.0.0:<port>, but the API server # # cert is only valid for 127.0.0.1, so TLS verification fails — this sed rewrites it to 127.0.0.1 to prevent that error.

# Linux + Docker Engine instead:
# clusterctl get kubeconfig dev > dev.kubeconfig
```

### 5. Install a CNI (kindnet)

Until a CNI is installed the nodes stay `NotReady`. We use **kindnet** — KIND's default
CNI. It is a single small image and reads each node's `spec.podCIDR` automatically, so it
works with the CAPD default pod CIDR (`192.168.0.0/16`) with no edits:

```
kubectl --kubeconfig=./dev.kubeconfig apply -f \
  https://raw.githubusercontent.com/kubernetes-sigs/kindnet/refs/heads/main/install-kindnet.yaml
```

> **Why not Calico here?** Calico works (its default pool also matches `192.168.0.0/16`),
> but it ships three large images plus three init-containers; on a slow link each image
> can take minutes to pull, *per cluster*. kindnet is one small image and comes up in
> seconds — the right trade-off for a local lab. Calico's extras (NetworkPolicy, BGP,
> eBPF) are production features we don't need here.

### 6. Verify `dev`

```
kubectl --kubeconfig=./dev.kubeconfig get nodes -o wide
```

Within a couple of minutes both nodes should be `Ready` and report **v1.34.0**. That's a
complete CAPI-managed cluster.

### 7. Add `test` and `prod` (same pattern)

Repeat the generate/apply for the other two environments so we have a fleet:

```
for ENV in test prod; do
  clusterctl generate cluster "$ENV" --flavor development \
    --kubernetes-version v1.34.0 \
    --control-plane-machine-count=1 \
    --worker-machine-count=1 \
    > "$ENV.yaml"
  kubectl apply -f "$ENV.yaml"
done

kubectl get clusters     # dev / test / prod, all managed from mgmt
```

Then install the CNI on each (and grab their kubeconfigs):

```
for ENV in test prod; do
  kind get kubeconfig --name "$ENV" > "$ENV.kubeconfig"
  sed -i '' 's#https://0.0.0.0:#https://127.0.0.1:#' "$ENV.kubeconfig"
  kubectl --kubeconfig="./$ENV.kubeconfig" apply -f \
    https://raw.githubusercontent.com/kubernetes-sigs/kindnet/refs/heads/main/install-kindnet.yaml
    
done
```

At this point three workload clusters exist, all at v1.34.0, all expressed as custom
resources inside the single `mgmt` cluster.

## 04 — Test: a centralized k8s version upgrade

This is the payoff of the whole repo. We take the fleet from **v1.34.0 -> v1.35.0** by
editing **one field per cluster** on the management cluster. CAPI does the rest: it
rolls the control plane, then the workers, replacing machines from a trusted image,
no SSH, no `kubeadm upgrade` on each node.

This is exactly the "3 clusters, upgraded manually" -> "clusters as CRDs, upgraded by
changing a field" jump.

### 0. Prep

Management operations run against the `mgmt` cluster — make sure `kubectl` points there:

```
kubectl config use-context kind-mgmt
kubectl get clusters     # dev / test / prod, all v1.34.0
```

Pre-pull the target node image on the host so the replacement machine containers start
fast (CAPD creates node containers from the host's image):

```
docker pull kindest/node:v1.35.0
```

> One-minor-at-a-time: you must always upgrade between Kubernetes minor versions in sequence
> (v1.34 -> v1.35, never v1.34 -> v1.36 in one hop).(https://cluster-api.sigs.k8s.io/tasks/upgrading-clusters). EKS enforces the same rule.

### 1. Upgrade `dev` (one field)

```
kubectl patch cluster dev --type merge \
  -p '{"spec":{"topology":{"version":"v1.35.0"}}}'
```

That's the entire upgrade command. Everything below is just watching CAPI execute it.

### 2. Watch the rolling replacement

```
clusterctl describe cluster dev
# in another shell:
kubectl get machines      # NAME ... PHASE AGE VERSION
```

What you'll observe, in order:

1. **Control plane first.** A **new** control-plane machine is created at v1.35.0, it
   joins, then the old v1.34.0 control-plane machine is deleted. The machine *name
   changes* — that's the tell that CAPI replaced it rather than upgrading it in place.
2. **Workers next.** Once the control plane is at v1.35.0, the MachineDeployment rolls
   the worker the same way: new v1.35.0 machine up, old one drained and deleted.

The high-level steps to fully upgrading a cluster are to first upgrade the control plane and then upgrade the worker machines.

> During the control-plane roll you'll briefly see **two** control-plane machines (the
> default keeps the old one until the new one is ready). That's one extra container for a
> minute — fine at 16 GB, but it's why we upgrade one cluster at a time.

### 3. Verify `dev`

Your existing `dev.kubeconfig` keeps working through the upgrade: the `dev-lb` endpoint
and the cluster CA are preserved across the control-plane replacement (unlike the reboot
case, which regenerated them). If needed, re-fetch with the usual fix:

```
kind get kubeconfig --name dev > dev.kubeconfig
sed -i '' 's#https://0.0.0.0:#https://127.0.0.1:#' dev.kubeconfig

kubectl --kubeconfig=./dev.kubeconfig get nodes -o wide   # both nodes v1.35.0
```

When both nodes report **v1.35.0**, `dev` is upgraded.


#### Why this works the way it does

Two ideas from the talks show up directly here.

**Version-skew guardrail (why control plane first).** CAPI refuses to upgrade workers
ahead of the control plane — a worker can't be a minor version *above* its control
plane. So even though you set one target version, CAPI sequences it safely: control
plane, then workers. You don't manage the ordering; the system does.

**Immutability (why machines are replaced, not edited).** CAPI treats a machine like an
immutable pod: to change it, it creates a new one from a trusted image and deletes the
old one. That's why the machine names changed. The benefit is no configuration drift —
every node is identical to its template — and predictable troubleshooting. The default
keeps availability during the roll (create the new machine before deleting the old).

> Aside (CAPI v1.12, Jan 2026): the project added **in-place updates** (apply
> low-disruption changes like an SSH key without replacing the machine) and **chained
> upgrades** (set a target several minors ahead and CAPI walks the steps for you). Our
> lab uses the default rolling replacement across a single minor, which is the safest and
> most common path.

### 4. Roll the fleet: `test`, then `prod`

Same one-field edit, one cluster at a time — verify each before moving on, the way you'd
sequence a real fleet (dev -> test -> prod):

```
# test
kubectl patch cluster test --type merge -p '{"spec":{"topology":{"version":"v1.35.0"}}}'
clusterctl describe cluster test
kubectl --kubeconfig=./test.kubeconfig get nodes -o wide   # v1.35.0, then continue

# prod
kubectl patch cluster prod --type merge -p '{"spec":{"topology":{"version":"v1.35.0"}}}'
clusterctl describe cluster prod
kubectl --kubeconfig=./prod.kubeconfig get nodes -o wide   # v1.35.0
```

Confirm the whole fleet:

```
kubectl get clusters     # dev / test / prod all VERSION v1.35.0
```

That's the centralized fleet upgrade: three clusters, three one-line edits, all driven
from the single management cluster.

### References

- The Cluster API Book — https://cluster-api.sigs.k8s.io
- [Introduction To Cluster API - Jussi Nummelin, Mirantis Inc](https://www.youtube.com/watch?v=UMNCKkdrfUo)
- [KubeCon 2026 Europe: In-place Updates with Cluster API: Fabrizio Pandini & Stefan Büringer (immutability, rollout strategies)](https://www.youtube.com/watch?v=CMf6rOPo9Z0)
