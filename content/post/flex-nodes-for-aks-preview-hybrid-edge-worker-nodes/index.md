---
title: 'Flex Nodes for AKS (Preview): Extending an Azure-Managed Control Plane to On-Premises and Edge Servers'
date: 2026-09-23T17:48:10+00:00
author: Mike Hacker
tags:
- App Modernization
- AI
- Announcements
- How To
categories:
- App Modernization
summary: 'A technical guide for government infrastructure teams on AKS flex nodes (public preview): how the architecture works, how it compares to Arc-enabled Kubernetes and AKS on Azure Local, CLI steps for onboarding and workload placement, and what to confirm before assuming availability in Azure Government.'
draft: false
image_prompt: Photorealistic editorial scene inside a roadside traffic signal equipment cabinet, door open at dusk, with a compact rack server installed neatly beside signal controllers and bundled fiber and copper cabling. Warm amber cabinet work light against the cool teal evening sky, with a blurred intersection and car light trails in the background. Single clear focal point on the server, life-size and grounded, suggesting local compute next to where data is created. No people, no text, no logos, no screens with readable content.
audio: audio.mp3
---

Most government IT shops run workloads in more than one place. One application runs in an Azure region. Another runs on servers in a traffic operations center, a water treatment facility, or a county data center, because the data it processes has to stay in that building or the latency to the cloud is too high. Until now, the usual Kubernetes answer was to build another cluster in each location. Each one brings its own control plane, upgrade cycle, identity setup, and policy drift.

On **September 22, 2026**, the AKS team [announced the public preview of flex nodes for AKS](https://blog.aks.azure.com/2026/09/22/flex-nodes-for-aks). Flex nodes let you join customer-managed virtual machines and bare metal hosts to an existing AKS cluster as worker nodes. That includes hosts in other Azure regions, on-premises, or at the edge. The control plane stays in Azure and is managed by Microsoft. The compute runs wherever you need it.

This post explains how the feature works, where it fits next to Azure Arc-enabled Kubernetes and AKS on Azure Local, how to onboard a node and control scheduling from the CLI, and what to check before you plan around it in Azure Government.

> **Preview status:** According to the [flex nodes overview on Microsoft Learn](https://learn.microsoft.com/azure/aks/flex-nodes-for-aks-overview) (updated September 2026), flex nodes are an AKS preview feature. They are not covered by an SLA and are "not meant for production use." Use them for evaluation, pilots, and labs.

## Architecture: How Flex Nodes Work

The [overview documentation](https://learn.microsoft.com/azure/aks/flex-nodes-for-aks-overview) describes five components:

| Component | Role |
|---|---|
| **User-managed host** | A VM or bare metal server you own that provides the compute and runs the agent |
| **Flex node agent** | Open-source software ([Azure/AKSFlexNode](https://github.com/Azure/AKSFlexNode)) that bootstraps the host, joins it to the cluster, and keeps it aligned with the desired state in AKS |
| **AKS cluster** | The Azure-managed control plane that schedules work across standard node pools and flex nodes |
| **ARM Machine resource** | Holds the authoritative desired state for each flex node, including its Kubernetes version |
| **AKS management APIs** | Handle lifecycle operations for the pool: upgrade, update, and removal |

The agent runs the Kubernetes worker inside an isolated environment on the host. The [project README](https://github.com/Azure/AKSFlexNode) describes it as a systemd-nspawn machine. Because of this, AKS can upgrade or reset the node without reimaging the physical server. The agent supports amd64 and arm64 hosts and detects NVIDIA GPUs automatically, configuring the container runtime for accelerated workloads.

### Identity

AKS didn't provision these hosts, so each one needs a verifiable identity. The [identity and access concepts article](https://learn.microsoft.com/azure/aks/flex-nodes-identity-access-concepts) lists three options:

- **Azure Arc managed identity:** for on-premises servers that are already Arc-enabled. It is secretless, which makes it the strongest option for most facility servers.
- **System- or user-assigned managed identity:** for flex nodes that are Azure VMs, for example capacity in another region.
- **Service principal:** for any other host. Microsoft recommends a certificate credential over a client secret.

Whichever option you choose, the identity is granted the **Azure Kubernetes Service Contributor Role** scoped to that one cluster, not the whole subscription. That limits what a compromised host identity can reach.

### Networking: Bring Your Own

This is the most important design point for the preview. Per the [networking concepts article](https://learn.microsoft.com/azure/aks/flex-nodes-networking-concepts), **AKS-managed network plugins (Azure CNI and Azure CNI Powered by Cilium) don't run on flex nodes during public preview.** You can attach flex nodes to a cluster that uses those plugins, but their pod networking and network policy capabilities won't extend to the flex nodes.

The documented path creates the cluster with `--network-plugin none` and installs **Unbounded-Net** on both AKS nodes and flex nodes. Unbounded-Net models each location as a *Site* with its own node and pod CIDRs, then routes pod traffic between Sites. The underlying Layer 3 path is your responsibility. The [planning guide](https://learn.microsoft.com/azure/aks/plan-flex-nodes-deployment) lists the options:

- **VNet peering:** for flex hosts in other Azure regions
- **Site-to-site VPN Gateway:** for on-premises facilities
- **ExpressRoute:** for agencies with dedicated private connectivity
- **Unbounded-Net WireGuard gateway:** an alternative topology, covered in the GitHub labs, for when no private routed path exists

Plan non-overlapping ranges for AKS nodes, AKS pods, flex nodes, flex pods, and Kubernetes services. The AKS control plane also needs a path back to each flex node's kubelet so that `kubectl logs`, `exec`, and `port-forward` work.

## Flex Nodes vs. Arc-Enabled Kubernetes vs. AKS on Azure Local

These three options solve different problems, and you can use them together.

| | **Flex nodes for AKS** | **Azure Arc-enabled Kubernetes** | **AKS on Azure Local** |
|---|---|---|---|
| Where the control plane runs | In Azure, managed by AKS | In your cluster, wherever it runs | On-premises, on your Azure Local infrastructure |
| What you bring | Individual Linux hosts (Ubuntu 24.04 or Azure Linux 3) | An existing CNCF-conformant cluster | Validated Azure Local hardware |
| Number of clusters | One cluster spanning locations | Many clusters, projected into Azure | One or more local clusters |
| Status (September 2026) | Public preview | Generally available | Generally available |
| Best fit | Pinning specific workloads to local hardware or borrowing remote capacity without running another control plane | Governing a mixed fleet with GitOps, Azure Policy, and Defender | Full on-premises Kubernetes that must keep operating on its own |

[Arc-enabled Kubernetes](https://learn.microsoft.com/azure/azure-arc/kubernetes/overview) gives you inventory, GitOps (Flux v2 or Argo CD), Azure Policy, Azure Monitor, and Microsoft Defender for Containers across clusters that run anywhere. [AKS on Azure Local](https://learn.microsoft.com/azure/aks/aksarc/aks-overview) runs the control plane locally through a management appliance. The [AKS platform comparison](https://learn.microsoft.com/azure/aks/aksarc/aks-platforms-compare) is a useful side-by-side reference.

Here is a practical way to decide. If the site must keep scheduling and self-healing workloads during a WAN outage, keep the control plane local with AKS on Azure Local. If the site has reliable connectivity and you mainly want a few workloads to run next to the data without another cluster to maintain, flex nodes remove a lot of operational overhead.

## CLI Walkthrough: Onboarding a Flex Node

The commands below come from the Microsoft Learn deployment series. They are shortened here, so follow the full articles for the environment file and validation steps.

### 1. Install the preview extension and register features

From [Prepare an AKS cluster for flex nodes](https://learn.microsoft.com/azure/aks/prepare-aks-cluster-for-flex-nodes). The minimum `aks-preview` version is **22.0.0b8**. The docs also say you need a subscription that is approved for the preview.

```bash
az extension add --name aks-preview --allow-preview true --upgrade

az feature register --namespace Microsoft.ContainerService --name AKSFlexNodePreview
az feature register --namespace Microsoft.ContainerService --name PutMachinePreview
az provider register --namespace Microsoft.ContainerService --wait
```

### 2. Create the cluster without a built-in network plugin

```bash
az aks create \
  --resource-group "${RESOURCE_GROUP}" \
  --name "${CLUSTER_NAME}" \
  --location "${LOCATION}" \
  --kubernetes-version "${AKS_VERSION}" \
  --nodepool-name systempool \
  --node-count 1 \
  --node-vm-size "${AKS_NODE_VM_SIZE}" \
  --network-plugin none \
  --pod-cidr "${AKS_POD_CIDR}" \
  --vnet-subnet-id "${AKS_SUBNET_ID}" \
  --service-cidr "${SERVICE_CIDR}" \
  --dns-service-ip "${DNS_SERVICE_IP}" \
  --enable-managed-identity \
  --ssh-access disabled --no-ssh-key
```

For a private API server, add `--enable-private-cluster --disable-public-fqdn --private-dns-zone system`. Your facility network must then be able to resolve and reach the private endpoint, for example through Azure DNS Private Resolver over VPN or ExpressRoute.

### 3. Install Unbounded-Net and define the Sites

From [Configure networking and create a flex node pool](https://learn.microsoft.com/azure/aks/configure-flex-nodes-networking):

```bash
kubectl unbounded install --timeout 5m

kubectl unbounded site init \
  --name flex-site \
  --cluster-node-cidr "${AKS_NODE_CIDR}" \
  --cluster-pod-cidr "${AKS_POD_CIDR}" \
  --node-cidr "${FLEX_NODE_CIDR}" \
  --pod-cidr "${FLEX_POD_CIDR}"
```

Then apply a `SitePeering` (`apiVersion: net.unbounded-cloud.io/v1alpha1`) that lists the `cluster` and `flex-site` Sites with `meshNodes: true`.

### 4. Create a flex node pool with a protective taint

```bash
az aks nodepool add \
  --resource-group "${RESOURCE_GROUP}" \
  --cluster-name "${CLUSTER_NAME}" \
  --name "${FLEX_POOL_NAME}" \
  --vm-set-type FlexNodes \
  --mode User \
  --kubernetes-version "${AKS_VERSION}" \
  --max-pods 75 \
  --max-unavailable 1 \
  --labels site=traffic-ops \
  --node-taints site=traffic-ops:NoSchedule
```

The `--vm-set-type FlexNodes` flag is what makes this a flex pool. The taint means only workloads that explicitly tolerate it can land on facility hardware.

### 5. Grant the host identity and bootstrap the host

From [Prepare a flex node host and identity](https://learn.microsoft.com/azure/aks/prepare-flex-node-host-identity), grant the role at cluster scope:

```bash
az role assignment create \
  --assignee-object-id "${HOST_PRINCIPAL_OBJECT_ID}" \
  --assignee-principal-type ServicePrincipal \
  --role "Azure Kubernetes Service Contributor Role" \
  --scope "${AKS_RESOURCE_ID}"
```

The host needs Ubuntu 24.04 LTS or Azure Linux 3, at least 4 vCPUs, and 8 GiB free under `/var/lib`. For an Arc-enabled server, [Attach a flex node to an AKS cluster](https://learn.microsoft.com/azure/aks/attach-flex-node-to-aks) runs the version-matched bootstrap script as root, after you download the release archive and verify its checksum:

```bash
bash /tmp/bootstrap.sh \
  --auth arc \
  --fetch-bootstrap-data \
  --cluster-resource-id "${AKS_RESOURCE_ID}" \
  --agent-pool-name "${FLEX_POOL_NAME}" \
  --agent-url "file:///tmp/${AGENT_ARCHIVE}" \
  --agent-sha256 "${AGENT_SHA256}" \
  --config-overrides "${CONFIG_OVERRIDES}"
```

For managed identity or service principal paths, you first run `az aks nodepool get-bootstrap-data` from your workstation. The bootstrap token it returns expires after one hour.

## Scheduling Data-Sensitive and AI Inference Workloads

Once the node joins, workload placement uses ordinary Kubernetes primitives. The [management article](https://learn.microsoft.com/azure/aks/manage-and-remove-flex-nodes) shows that every flex node carries the label `kubernetes.azure.com/nodepool-type=FlexNodes`. You can add site-specific labels per machine:

```bash
az aks machine update \
  --resource-group "${RESOURCE_GROUP}" \
  --cluster-name "${CLUSTER_NAME}" \
  --nodepool-name "${FLEX_POOL_NAME}" \
  --machine-name "${FLEX_MACHINE_NAME}" \
  --labels site=traffic-ops data-class=local-only \
  --node-taints site=traffic-ops:NoSchedule
```

This command **replaces** the labels and taints it manages, so include every value you want the machine to keep. Next, pin an inference service that processes raw roadway sensor or camera feeds to that hardware:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: intersection-inference
spec:
  replicas: 1
  selector:
    matchLabels: { app: intersection-inference }
  template:
    metadata:
      labels: { app: intersection-inference }
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.azure.com/nodepool-type
                operator: In
                values: ["FlexNodes"]
              - key: data-class
                operator: In
                values: ["local-only"]
      tolerations:
      - key: site
        operator: Equal
        value: traffic-ops
        effect: NoSchedule
      containers:
      - name: model-server
        image: <your-registry>/intersection-model:1.0
```

This gives you a clean split:

- **At the facility:** high-volume raw data, such as video frames, sensor telemetry, or building management system signals, is processed where it is created. Only derived results, like counts, anomaly events, or occupancy summaries, leave the building.
- **In Azure:** dashboards, APIs, model training, and storage of aggregated results run on standard node pools with Azure CNI, autoscaling, and zone redundancy.

For GPU inference, the agent's NVIDIA detection prepares the container runtime. Validate in your lab how GPUs are advertised to the scheduler, for example with a device plugin, before you write `nvidia.com/gpu` requests into manifests. If a site needs full local autonomy for inference, the AKS team's September 2026 [AI Inference on AKS enabled by Azure Arc](https://blog.aks.azure.com/) series covers that pattern.

### Operational guardrails

- A flex node can't run a Kubernetes version newer than the control plane. Upgrade the control plane first with `az aks upgrade --control-plane-only`, then test one machine, then run `az aks nodepool upgrade`.
- You **can't stop** a cluster that contains a flex node pool, and the Azure portal can't create or manage flex pools ([support policy](https://learn.microsoft.com/azure/aks/flex-nodes-support-policy)).
- You own the host OS, patching, firmware, routing, firewalls, and Unbounded-Net lifecycle. Microsoft owns the control plane and the flex node integration.

## Azure Commercial vs. Azure Government: What to Check

As of this writing, the flex nodes Learn documentation (September 2026) does **not** state availability in Azure Government. The walkthroughs assume the commercial cloud; for example, the rendered agent configuration uses `https://management.azure.com` as the Resource Manager endpoint. Azure Government uses different endpoints, such as `management.usgovcloudapi.net` for ARM and `login.microsoftonline.us` for Entra ID, as listed in [Compare Azure Government and global Azure](https://learn.microsoft.com/azure/azure-government/compare-azure-government-global-azure). Don't assume that editing an endpoint makes an unsupported configuration work.

Before you plan a Government-cloud pilot:

```bash
az cloud set --name AzureUSGovernment
az login
az feature show --namespace Microsoft.ContainerService --name AKSFlexNodePreview --query properties.state
```

Also check [Products available by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/), confirm with your Microsoft account team, and watch the [AKSFlexNode releases](https://github.com/Azure/AKSFlexNode/releases). Agent v0.2.0 shipped in September 2026. For workloads that must stay in Azure Government today, AKS in Azure Government, Arc-enabled Kubernetes, or AKS on Azure Local are the established options. A commercial-cloud flex node lab is a low-risk way to test the architecture in the meantime.

## Why This Matters for Government

**For CxOs:** Flex nodes offer one Kubernetes operating model across the cloud and your own buildings. Agencies can put existing on-premises servers to work, keep regulated data physically local, and avoid paying for extra control planes, upgrade cycles, and separate policy baselines. The AKS team named the public sector explicitly as a data-residency scenario in the September 2026 announcement.

**For IT leaders and platform engineers:** You get one `kubectl` context, one RBAC model, and one upgrade path through AKS APIs. Placement uses the node affinity and taints your teams already know. Secretless Arc identities scoped to a single cluster fit zero-trust programs. The shared-responsibility boundaries are documented clearly.

**For developers:** An inference service for a transportation, public works, or facilities use case deploys with the same manifest style and CI/CD pipeline as the rest of the application. Only a node selector and a toleration change.

**Recommended next step:** Build a lab in Azure Commercial with an Arc-enabled server in a non-production facility, connected over your existing site-to-site VPN. Validate DNS, kubelet callbacks, cross-Site pod traffic, and upgrade behavior. Record what you learn, and send feedback through the [AKSFlexNode issue tracker](https://github.com/Azure/AKSFlexNode) so the product can mature around public-sector requirements before general availability.
