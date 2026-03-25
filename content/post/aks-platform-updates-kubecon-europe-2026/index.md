---
title: 'AKS Platform Updates: What Container Teams Need to Know from KubeCon Europe 2026'
date: 2026-03-25T12:53:31+00:00
author: Mike Hacker
tags:
- App Modernization
- AI
- Announcements
categories:
- App Modernization
summary: A roundup of the most impactful AKS announcements from KubeCon Europe 2026, including network metrics filtering, GPU observability, cross-cluster networking, Elastic SAN storage, and AI-powered troubleshooting for government container platform teams.
draft: false
image_prompt: An isometric illustration of interconnected Azure Kubernetes clusters with visual elements representing GPU monitoring dashboards, network traffic flows, storage volumes, and an AI assistant interface, using a modern government technology aesthetic with blue and teal tones
image: cover.jpg
audio: audio.mp3
---

At KubeCon + CloudNativeCon Europe 2026 in Amsterdam, Microsoft announced a wave of Azure Kubernetes Service (AKS) enhancements that span observability, multi-cluster operations, storage, and AI-assisted troubleshooting. For government IT teams running containerized workloads on AKS, these updates address persistent operational challenges: controlling monitoring costs at scale, gaining visibility into GPU workloads, connecting services across clusters, simplifying stateful storage, and accelerating incident response.

This post breaks down the five most relevant announcements and what they mean for public sector container platform teams.

## Container Network Metrics Filtering: Control Your Observability Costs

Advanced Container Networking Services (ACNS) already provides deep network visibility through node-level and pod-level metrics, capturing traffic volume, dropped packets, TCP state, and DNS resolution times using eBPF technology. The new **container network metrics filtering** capability, currently in preview, takes this a step further by letting operators filter metrics at the source before they are collected and stored.

Using Kubernetes custom resources, platform teams can define exactly which metrics matter for their environments. Filtering supports multiple dimensions, including namespace-based filtering to focus on specific applications, pod and label-based filtering for targeted monitoring, and metric-specific filtering to collect only essential data types.

For government organizations, this addresses a real operational tension. You need comprehensive observability for compliance and security, but the sheer volume of metrics from large clusters can inflate Azure Monitor costs and slow down dashboards. Metrics filtering lets teams maintain deep visibility where it counts while reducing noise and storage costs everywhere else.

This feature requires the Cilium data plane (Kubernetes 1.29+) and integrates with Azure managed service for Prometheus and Azure Managed Grafana.

**Learn more:** [Container Network Metrics Filtering](https://learn.microsoft.com/en-us/azure/aks/container-network-observability-metrics?tabs=Cilium#container-network-metrics-filtering-preview) | [Advanced Container Networking Services Overview](https://learn.microsoft.com/en-us/azure/aks/advanced-container-networking-services-overview)

## GPU Metrics in Azure Monitor: Close the AI Workload Visibility Gap

As government agencies explore AI and machine learning workloads on Kubernetes, a critical monitoring blind spot has been GPU utilization. Previously, getting GPU telemetry into your monitoring stack required manual exporter configuration and custom integration work.

AKS now [surfaces GPU performance and utilization metrics](https://aka.ms/aks/managed-gpu-metrics) directly into managed Prometheus and Grafana. This means GPU telemetry lives in the same monitoring stack teams already use for CPU, memory, and network metrics, enabling unified capacity planning, alerting, and cost allocation.

For agencies running inference workloads, batch processing, or evaluating AI pilots on AKS, this removes the need for separate GPU monitoring tools. Platform teams can set alerts on GPU utilization thresholds, track GPU usage trends for budget forecasting, and ensure expensive GPU node pools are used efficiently.

This is especially valuable for organizations that need to justify GPU infrastructure spend. Having GPU utilization data alongside standard Kubernetes metrics in Grafana dashboards makes it straightforward to demonstrate value and optimize resource allocation.

## Cross-Cluster Networking in Fleet Manager: Unified Multi-Cluster Connectivity

Government organizations frequently operate multiple AKS clusters for environment isolation, regional redundancy, or compliance boundary separation. Historically, connecting services across these clusters required custom plumbing, inconsistent service discovery, and limited cross-cluster visibility.

Azure Kubernetes Fleet Manager now offers [cross-cluster networking](https://aka.ms/kubernetes-fleet/networking/cross-cluster) through a managed Cilium cluster mesh. This provides:

- **Unified connectivity** across AKS clusters without manual network configuration
- **A global service registry** for cross-cluster service discovery
- **Intelligent routing** with configuration managed centrally rather than duplicated per cluster

Fleet Manager already supports resource propagation through the `ClusterResourcePlacement` API, enabling platform administrators to deploy Kubernetes resources to selected member clusters using policies like `PickAll`, `PickFixed`, or `PickN` with affinity rules and topology spread constraints. Cross-cluster networking complements this by ensuring that services deployed across clusters can actually find and communicate with each other.

For multi-agency shared platforms or disaster recovery architectures, this means services in one cluster can transparently discover and route to services in another. Combined with Fleet Manager's [multi-cluster load balancing](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/l4-load-balancing) using `ServiceExport` and `MultiClusterService` resources, agencies can build resilient architectures that span clusters and regions with significantly less operational overhead.

**Learn more:** [Fleet Manager Overview](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/) | [Resource Propagation Concepts](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/concepts-resource-propagation)

## Azure Container Storage with Elastic SAN: Scalable Stateful Workloads

Stateful workloads on Kubernetes have always been a challenge, and government agencies running databases, analytics engines, or records management systems on AKS feel this acutely. Azure Container Storage now supports [Elastic SAN as a backend storage option](https://learn.microsoft.com/en-us/azure/storage/container-storage/use-container-storage-with-elastic-san), offering durable, network-attached block storage that scales with your applications.

Key benefits of the Elastic SAN integration include:

- **Predictable throughput** from a centralized capacity pool distributed across volumes
- **Built-in redundancy** with locally redundant storage (LRS) and zone-redundant storage (ZRS) options
- **High volume density** with the ability to provision and attach thousands of persistent volumes per cluster, avoiding Azure Resource Manager disk attachment limits (such as 64 disks per VM)
- **Faster attach/detach operations** using NVMe over Fabrics (NVMe-oF) or iSCSI protocols
- **Kubernetes-native management** through standard StorageClass and PersistentVolumeClaim resources

Azure Container Storage supports three provisioning models for Elastic SAN: fully dynamic provisioning, pre-provisioned Elastic SAN with volume groups, and static provisioning where administrators precreate all resources. This flexibility allows government teams to choose the model that best fits their change management and governance requirements.

Instead of provisioning and managing individual disks per workload, clusters consume storage from a shared Elastic SAN pool. This simplifies capacity planning for stateful workloads with variable demands and reduces provisioning overhead at scale.

**Learn more:** [Azure Container Storage Introduction](https://learn.microsoft.com/en-us/azure/storage/container-storage/container-storage-introduction) | [Use Container Storage with Elastic SAN](https://learn.microsoft.com/en-us/azure/storage/container-storage/use-container-storage-with-elastic-san)

## AI-Powered Container Networking Troubleshooting: From Symptoms to Solutions Faster

Network troubleshooting in Kubernetes is notoriously time-consuming. When a service cannot reach a dependency or latency spikes appear, operators typically need to correlate metrics, inspect flow logs, review network policies, and trace traffic paths manually. This process demands deep expertise and can take hours.

The new [agentic container networking](https://learn.microsoft.com/en-us/azure/aks/advanced-container-networking-services-overview) capability in Advanced Container Networking Services introduces a web-based interface that accepts natural-language queries and translates them into read-only diagnostics using live telemetry. Rather than requiring operators to know which Prometheus queries to run or which Hubble flow logs to inspect, the AI agent interprets questions like "why can't my frontend pods reach the payment service?" and surfaces relevant diagnostics automatically.

This is a read-only diagnostic tool, meaning it cannot make changes to your cluster. It shortens the path from "something is wrong" to "here is what to investigate" without introducing risk. For government teams that may have limited Kubernetes networking expertise on staff, this lowers the barrier to effective incident response.

This capability builds on the broader ACNS observability stack, including container network metrics and container network logs that capture L3, L4, and supported L7 traffic (HTTP, gRPC, Kafka) with metadata like source and destination IPs, pod names, namespaces, ports, protocols, and policy verdicts.

## Why This Matters for Government

These AKS updates collectively address the operational maturity challenges that government container platform teams face daily:

- **Cost governance:** Metrics filtering gives agencies direct control over observability costs without sacrificing visibility. This is critical when IT budgets face annual scrutiny and per-cluster monitoring costs need clear justification.
- **AI readiness:** Built-in GPU metrics remove a barrier to running AI workloads responsibly on AKS. Agencies exploring AI pilots can monitor resource utilization and demonstrate ROI with the same tools they use for all other workloads.
- **Resilience and continuity:** Cross-cluster networking through Fleet Manager enables multi-cluster architectures that support disaster recovery and business continuity requirements common in government compliance frameworks.
- **Data sovereignty and storage compliance:** Elastic SAN integration with zone-redundant storage options helps agencies meet data durability and availability requirements for sensitive workloads while simplifying storage management.
- **Operational efficiency:** AI-powered troubleshooting reduces mean time to resolution and lowers the expertise barrier for network incident response, which is especially valuable for agencies with lean platform engineering teams.

> **Azure Government note:** When evaluating these features, confirm availability in Azure Government regions for workloads that require it. AKS features typically reach Azure commercial first, with Azure Government availability following. Check the [Azure Government services availability](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) page and the [AKS release tracker](https://github.com/Azure/AKS/releases) for the latest status. Features in preview may not yet be available in Azure Government.

## Getting Started

For teams already running AKS clusters, most of these capabilities can be enabled on existing clusters:

- **ACNS features** (metrics filtering, network logs, agentic diagnostics): Enable with `az aks update --enable-acns` on Cilium-based clusters. See the [setup guide](https://learn.microsoft.com/en-us/azure/aks/use-advanced-container-networking-services).
- **GPU metrics**: Create a [managed GPU-enabled node pool](https://aka.ms/aks/managed-gpu-metrics) (preview) to automatically collect GPU performance and utilization metrics via the DCGM exporter.
- **Fleet Manager networking**: Requires a Fleet resource with member clusters. Start with the [Fleet Manager quickstart](https://learn.microsoft.com/en-us/azure/kubernetes-fleet/).
- **Container Storage with Elastic SAN**: Install Azure Container Storage on your AKS cluster and create an Elastic SAN-backed StorageClass. See the [installation guide](https://learn.microsoft.com/en-us/azure/storage/container-storage/install-container-storage-aks).

These announcements were shared at [KubeCon + CloudNativeCon Europe 2026](https://opensource.microsoft.com/blog/2026/03/24/whats-new-with-microsoft-in-open-source-and-kubernetes-at-kubecon-cloudnativecon-europe-2026/) by Brendan Burns, Corporate Vice President and Technical Fellow at Microsoft. For government teams investing in container platforms, this wave of updates signals that AKS continues to mature in the areas that matter most: operational control, security, observability, and multi-cluster management at scale.
