---
title: 'Cross-Cluster Networking with Azure Kubernetes Fleet Manager: A Cilium-Based Deep Dive'
date: 2026-05-27T16:15:42+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
- Security
categories:
- App Modernization
summary: How to design resilient multi-cluster, multi-region AKS architectures for government workloads using Azure Kubernetes Fleet Manager, Azure CNI Powered by Cilium, and the ServiceExport / MultiClusterService pattern.
draft: false
image_prompt: A cinematic wide-angle photograph of two massive interconnected industrial rope bridges suspended high over a misty canyon at golden hour, with hundreds of taut steel cables crossing between them in a precise woven pattern, dramatic side lighting catching the metal strands, deep atmospheric haze in the canyon below. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
audio: audio.mp3
---

Government platform teams running Kubernetes at scale rarely operate a single cluster anymore. Regional resiliency requirements, blast-radius containment, regulated-vs-unregulated workload separation, and the simple reality that an AKS control plane upgrade should never be a citizen-facing outage all push organizations toward a *fleet* of clusters. The hard part has always been the networking layer: how do pods in one cluster reliably reach pods in another, how do you fail traffic over between regions, and how do you enforce policy consistently when each cluster has its own CNI state?

This post walks through the current state of cross-cluster networking on Azure as of mid-2026, anchored on three building blocks: **Azure Kubernetes Fleet Manager**, **Azure CNI Powered by Cilium**, and **Advanced Container Networking Services (ACNS)**. The goal is a practical architecture pattern you can apply to multi-region AKS workloads in either Azure commercial or Azure US Government.

## The building blocks

### Fleet Manager, hub clusters, and member clusters

Azure Kubernetes Fleet Manager groups AKS clusters into a single managed resource. When you create a Fleet Manager *with a hub cluster*, you get a control surface for multi-cluster scheduling, Kubernetes resource propagation (`ClusterResourcePlacement` / `ResourcePlacement`), staged updates, and managed multi-tenant namespaces. Member clusters can live in different regions, resource groups, or subscriptions, as long as they share a Microsoft Entra tenant. Per the Fleet Manager concepts documentation (last updated April 2026), member cluster registration creates a `MemberCluster` resource on the hub that you can label and taint to drive placement decisions ([Microsoft Learn: Fleets and member clusters](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/concepts-fleet)).

### Azure CNI Powered by Cilium

Underneath, the cluster networking that makes cross-cluster connectivity work cleanly is Azure CNI Powered by Cilium. This combines Azure's CNI control plane with Cilium's eBPF dataplane, removes `kube-proxy`, and gives you native L3/L4 `CiliumNetworkPolicy` and `CiliumClusterwideNetworkPolicy` without installing a separate policy engine. The current support matrix (Microsoft Learn, updated for AKS 1.34/1.35 with Cilium 1.18.6) covers overlay, VNet-integrated, and node-subnet IPAM modes ([Azure CNI Powered by Cilium](https://learn.microsoft.com/en-us/azure/aks/azure-cni-powered-by-cilium)).

Layer 7 policies, FQDN filtering, flow logs, WireGuard transparent encryption, and mTLS are gated behind **Advanced Container Networking Services (ACNS)** - an add-on you opt into per cluster. For government workloads where in-transit encryption inside the VNet is a meaningful control, WireGuard via ACNS is the cleanest option because it does not require sidecars or service mesh injection.

### Multi-cluster Layer 4 load balancing

Fleet Manager's multi-cluster L4 load balancing is currently in **public preview** (docs last updated May 21, 2026). It introduces three Kubernetes resources that, together, give you cross-cluster service discovery and traffic spreading:

- `ServiceExport` - declared on a member cluster (or propagated from the hub), marks a Service as eligible for fleet-wide exposure.
- `ServiceImport` - automatically created on the hub and all other members to represent the exported service.
- `MultiClusterService` - the user-authored object that configures an Azure Load Balancer in each member cluster to distribute incoming traffic across the service's endpoints in every cluster that has exported it.

A hard prerequisite called out in the docs: target clusters must use **Azure CNI** so that pod IPs are directly addressable on the VNet and routable from the Azure Load Balancer ([Multi-cluster L4 load balancing](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/concepts-l4-load-balancing)). Overlay-mode pod IPs are not load-balancer-routable across clusters, so plan the IPAM choice up front.

## A reference architecture for a multi-region government workload

Imagine a permitting application that must remain available if an entire Azure region goes dark. A workable layout:

- **Fleet Manager hub** in one region, configured with a hub cluster.
- **Member cluster A** - AKS in US Gov Virginia (or East US 2 for commercial), Azure CNI with Cilium, dynamic pod IP from a dedicated pod subnet.
- **Member cluster B** - AKS in US Gov Arizona (or Central US), identical CNI configuration, peered VNets.
- **ACNS enabled** on both clusters for FQDN egress filtering, flow logs, and WireGuard pod-to-pod encryption.
- **Resource propagation** via `ClusterResourcePlacement` from the hub for the namespace, Deployment, Service, and `ServiceExport`.
- **MultiClusterService** authored on the hub and propagated to both members so each region can take ingress and steer traffic across the fleet.
- **Azure Front Door** in front of the two regional ingress points for global anycast and health-probed failover.

## Building it: the commands and manifests

### Create CNI-Cilium clusters with a dedicated pod subnet

```bash
az aks create \
  --name aks-fleet-east \
  --resource-group rg-fleet \
  --location eastus2 \
  --max-pods 250 \
  --network-plugin azure \
  --vnet-subnet-id $NODE_SUBNET_ID \
  --pod-subnet-id $POD_SUBNET_ID \
  --network-dataplane cilium \
  --generate-ssh-keys
```

Use VNet-integrated pod IPs (Option 2 in the AKS CNI-Cilium docs) for any cluster that will participate in `MultiClusterService` so that pod IPs are addressable from the Azure Load Balancer.

### Join clusters to the fleet

```bash
az fleet create -g rg-fleet -n contoso-fleet --enable-hub
az fleet member create -g rg-fleet --fleet-name contoso-fleet \
  --name east --member-cluster-id $AKS_EAST_ID
az fleet member create -g rg-fleet --fleet-name contoso-fleet \
  --name central --member-cluster-id $AKS_CENTRAL_ID
```

Add labels on the member resources (region, environment, data-classification) via `az fleet member update` so that placement policies can target them.

### Propagate the workload from the hub

```yaml
apiVersion: placement.kubernetes-fleet.io/v1
kind: ClusterResourcePlacement
metadata:
  name: permits-api
spec:
  resourceSelectors:
    - group: ""
      kind: Namespace
      name: permits
      version: v1
  policy:
    placementType: PickAll
```

A `PickAll` policy with a label selector on `env=prod` is a clean way to ensure every production member receives the namespace and its contents.

### Export and aggregate the Service

On the hub (propagated via the placement above):

```yaml
apiVersion: networking.fleet.azure.com/v1alpha1
kind: ServiceExport
metadata:
  namespace: permits
  name: permits-api
---
apiVersion: networking.fleet.azure.com/v1alpha1
kind: MultiClusterService
metadata:
  namespace: permits
  name: permits-api
spec:
  serviceImport:
    name: permits-api
```

Fleet Manager will materialize a `ServiceImport` on every other member and wire each cluster's Azure Load Balancer to forward to pod endpoints across the fleet.

### Enforce policy with Cilium

Because `kube-proxy` is gone and identities are eBPF-based, you can write a single `CiliumClusterwideNetworkPolicy` on the hub and propagate it everywhere:

```yaml
apiVersion: cilium.io/v2
kind: CiliumClusterwideNetworkPolicy
metadata:
  name: permits-api-ingress
spec:
  endpointSelector:
    matchLabels:
      app: permits-api
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: permits-web
      toPorts:
        - ports:
            - port: "8443"
              protocol: TCP
```

With ACNS enabled, you can add a `CiliumNetworkPolicy` with `toFQDNs` rules to constrain egress to a known list of state interchange endpoints.

## Architecture patterns worth knowing

**Active-active across paired regions.** `MultiClusterService` distributes evenly to healthy endpoints in any member with the service exported. Pair with Azure Front Door for global routing and you get regional failover that is automatic from the citizen's perspective.

**Active-passive with placement weights.** Use `PickN` placement policies and topology spread constraints to bias workloads toward a primary region but keep warm capacity in a secondary. Failover becomes a label change on the hub rather than a runbook.

**Tenant isolation with managed fleet namespaces.** For shared infrastructure clusters that serve multiple agencies, managed fleet namespaces give each tenant a quota-bound slice without granting cluster-admin to the underlying AKS resources.

**Encryption in depth.** WireGuard via ACNS encrypts pod-to-pod traffic on top of whatever NSG and Azure Firewall controls you already have at the VNet boundary - a useful belt-and-suspenders posture for FedRAMP and CJIS workloads.

## Why This Matters for Government

State, county, and city IT shops face a particular cocktail of requirements: regional resilience, predictable maintenance windows, in-transit encryption, strict tenant separation, and audit-friendly policy. A single big AKS cluster cannot serve all of those goals at once. Fleet Manager turns a collection of smaller, purpose-fit clusters into a single managed surface, and Cilium's eBPF dataplane makes per-cluster networking both faster and more observable without forcing you to adopt a heavyweight service mesh.

Two deployment notes specific to public sector customers:

1. **Azure US Government availability.** Fleet Manager's L4 multi-cluster load balancing is in preview. Preview features are explicitly excluded from SLAs and are not meant for production use; treat them as an architecture validation path while you plan for GA. Always confirm current regional availability for Azure Government on the [Products available by region](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/) page and against the [Azure Government compare guide](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure).
2. **GCC tenants.** Most readers operate in M365 GCC. Fleet Manager itself is an Azure resource and is unaffected by GCC vs GCC High distinctions in M365; what matters is the Azure cloud your AKS clusters live in (commercial vs US Gov). Identity, however, must terminate in the correct Entra tenant for member registration.

## Where to go next

- [Azure Kubernetes Fleet Manager - Fleets and member clusters](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/concepts-fleet) (Microsoft Learn, April 2026)
- [Multi-cluster Layer 4 load balancing (preview)](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/concepts-l4-load-balancing) (Microsoft Learn, May 2026)
- [Intelligent resource placement with Fleet Manager](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/concepts-resource-propagation) (Microsoft Learn)
- [Configure Azure CNI Powered by Cilium](https://learn.microsoft.com/en-us/azure/aks/azure-cni-powered-by-cilium) (Microsoft Learn)

If you are standing up your first fleet, start small: two clusters in two regions, a single non-critical workload, Cilium policies in audit mode, and an active-active `MultiClusterService` behind Front Door. Once the operational patterns feel familiar, scale the fleet horizontally rather than scaling any single cluster vertically. That is the architectural shift that makes resilient government Kubernetes feasible.
