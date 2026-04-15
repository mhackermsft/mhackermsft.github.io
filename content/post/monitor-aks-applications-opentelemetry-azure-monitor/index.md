---
title: 'Monitor AKS Applications with OpenTelemetry and Azure Monitor: A Hands-On Guide'
date: 2026-04-14T20:50:50+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
- Security
categories:
- Azure
summary: Walk through the new public preview that brings OpenTelemetry-based autoinstrumentation to AKS, covering setup, configuration, and observability best practices for government container workloads.
draft: false
image_prompt: A dramatic low-angle shot of a weathered stone lighthouse on a rocky cliff at dusk, its powerful beam cutting through dense fog to illuminate a harbor filled with neatly stacked shipping containers in deep teals and rusted oranges. Thin golden light trails arc from the lighthouse beam down to individual containers, as if tracing connections between them. The moody sky glows with deep purple and amber tones reflected on wet rocks below. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
audio: audio.mp3
---

As government agencies move more mission-critical workloads into containers on Azure Kubernetes Service (AKS), observability becomes non-negotiable. You need to know when a citizen-facing application is slow, why a background job is failing, and where latency is hiding across distributed microservices. Historically, that meant manually instrumenting every service with SDKs, which is tedious for large portfolios and error-prone across polyglot codebases.

Microsoft has addressed this gap with **AKS autoinstrumentation for Azure Monitor Application Insights**, now in [public preview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/kubernetes-codeless). This feature injects the [Azure Monitor OpenTelemetry Distro](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable) into your application pods automatically, generating distributed traces, metrics, and logs without requiring any code changes. In this post, we will walk through the architecture, step-by-step setup, and configuration patterns that government IT teams should know.

## What Is OpenTelemetry and Why Does It Matter?

[OpenTelemetry](https://opentelemetry.io/) (OTel) is a vendor-neutral, open-source observability framework backed by the Cloud Native Computing Foundation (CNCF). It standardizes how applications emit traces, metrics, and logs so that you are not locked into any single monitoring vendor. Azure Monitor's OpenTelemetry Distro builds on this foundation, adding Azure-specific exporters and configuration that route telemetry directly into Application Insights.

For government organizations, this matters because:

- **Vendor neutrality** reduces procurement risk. OpenTelemetry instrumentation works with multiple backends.
- **Standardized telemetry** means consistent observability across Java, Node.js, .NET, and Python services regardless of which team built them.
- **Reduced code changes** lower the risk of introducing bugs when adding monitoring to production workloads.

## Architecture Overview

The autoinstrumentation feature works by deploying a mutating admission webhook into your AKS cluster. When a pod starts (or restarts), the webhook intercepts the pod spec and injects the appropriate OpenTelemetry Distro sidecar based on the language you configure. The injected agent collects telemetry and exports it to your Application Insights resource via its connection string.

Here is how the pieces fit together within the broader AKS monitoring stack:

- **Application-level telemetry** (traces, dependencies, exceptions, logs) flows from the OTel Distro into Application Insights
- **Infrastructure metrics** (CPU, memory, pod counts) flow from [Azure Monitor managed service for Prometheus](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-metrics-overview) into an Azure Monitor workspace
- **Container logs** (stdout/stderr) flow from the Azure Monitor Agent into a Log Analytics workspace via [Container insights](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable)
- **Visualization** ties it all together through [Azure Managed Grafana](https://learn.microsoft.com/en-us/azure/managed-grafana/overview) dashboards

This layered approach gives you full-stack visibility: from the Kubernetes control plane down to individual HTTP requests inside your application code.

## Prerequisites

Before you begin, ensure you have:

- An AKS cluster running a Kubernetes deployment using **Java** or **Node.js** (the two languages supported in this preview)
- A [workspace-based Application Insights resource](https://learn.microsoft.com/en-us/azure/azure-monitor/app/create-workspace-resource)
- Azure CLI version 2.60.0 or later
- At least **Contributor** access to the cluster

> **Important:** This preview is currently incompatible with Windows node pools and Linux Arm64 node pools. Plan your node pool architecture accordingly.

## Step-by-Step Setup

### 1. Install the AKS Preview Extension

The autoinstrumentation feature requires the `aks-preview` CLI extension:

```bash
# Install the preview extension
az extension add --name aks-preview

# Or update if already installed
az extension update --name aks-preview

# Verify CLI version meets the 2.60.0+ requirement
az version
```

### 2. Register the Feature Flag

Register the `AzureMonitorAppMonitoringPreview` feature flag in your subscription. Note that registration can take up to several hours to propagate:

```bash
# Register the feature flag
az feature register \
  --namespace "Microsoft.ContainerService" \
  --name "AzureMonitorAppMonitoringPreview"

# Check registration status (wait for "Registered")
az feature list -o table \
  --query "[?contains(name, 'Microsoft.ContainerService/AzureMonitorAppMonitoringPreview')].{Name:name,State:properties.state}"

# Re-register the provider once the feature is registered
az provider register --namespace "Microsoft.ContainerService"
```

### 3. Prepare the Cluster

You can enable application monitoring during cluster creation or on an existing cluster.

**During cluster creation:**

```bash
az aks create \
  --resource-group myGovResourceGroup \
  --name myAKSCluster \
  --enable-azure-monitor-app-monitoring \
  --generate-ssh-keys
```

**On an existing cluster via the Azure portal:**

1. Navigate to your AKS cluster in the Azure portal
2. Select the **Monitor** pane
3. Check the **Enable application monitoring** box
4. Select **Review + enable**

### 4. Onboard Your Deployments

You have two onboarding approaches: **namespace-wide** (instrument everything in a namespace) or **per-deployment** (selective instrumentation with different Application Insights resources).

#### Namespace-Wide Onboarding

This is the simplest path. From the Azure portal, navigate to the **Namespaces** pane, select a namespace, choose **Application Monitoring**, pick your languages (Java, Node.js), and select **Configure**.

#### Per-Deployment Onboarding with Custom Resources

For more granular control, create an `Instrumentation` custom resource (CR) for each configuration scenario:

```yaml
apiVersion: monitor.azure.com/v1
kind: Instrumentation
metadata:
  name: citizen-portal-instrumentation
  namespace: citizen-services
spec:
  settings:
    autoInstrumentationPlatforms: []
  destination:
    applicationInsightsConnectionString: "InstrumentationKey=<your-key>;IngestionEndpoint=https://<region>.in.applicationinsights.azure.com/"
```

Then annotate each deployment to associate it with the CR:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: citizen-portal-api
  namespace: citizen-services
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-java: "citizen-portal-instrumentation"
    spec:
      containers:
        - name: api
          image: myregistry.azurecr.io/citizen-portal-api:latest
```

For Node.js services, use `instrumentation.opentelemetry.io/inject-nodejs` instead.

### 5. Restart Deployments

Autoinstrumentation takes effect only after a pod restart:

```bash
kubectl rollout restart deployment citizen-portal-api -n citizen-services
```

After the restart, generate some traffic against your application and navigate to your Application Insights resource. Within a few minutes, you should see distributed traces, dependency maps, and performance metrics populating the portal.

## Enabling Application Logs in Application Insights

By default, container stdout/stderr logs go to Container Insights. You can optionally route application logs into Application Insights as well, which provides correlated logs alongside distributed traces. This is especially valuable for microservices that use structured logging frameworks rather than simple console output.

Add this annotation to your deployment:

```yaml
spec:
  template:
    metadata:
      annotations:
        monitor.azure.com/enable-application-logs: "true"
        instrumentation.opentelemetry.io/inject-java: "true"
```

> **Cost consideration:** Enabling logs in both Container Insights and Application Insights creates duplication. Evaluate whether you need both, or if one source can serve your teams. You can [filter container log collection with ConfigMap](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-data-collection-configmap) to reduce overlap.

## Completing the Observability Stack

Autoinstrumentation handles application-level telemetry, but a production AKS cluster needs the full monitoring stack. Here is how to enable the other layers alongside autoinstrumentation:

```bash
# Enable Prometheus metrics and link to Grafana
az aks update \
  --name myAKSCluster \
  --resource-group myGovResourceGroup \
  --enable-azure-monitor-metrics \
  --azure-monitor-workspace-resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Monitor/accounts/<workspace> \
  --grafana-resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.Dashboard/grafana/<grafana-name>

# Enable Container Insights for log collection
az aks enable-addons \
  --addon monitoring \
  --name myAKSCluster \
  --resource-group myGovResourceGroup \
  --workspace-resource-id /subscriptions/<sub>/resourceGroups/<rg>/providers/Microsoft.OperationalInsights/workspaces/<la-workspace>
```

With all three layers enabled, you get:

| Layer | Tool | Data |
|-------|------|------|
| Application | Application Insights (OTel Distro) | Traces, dependencies, exceptions, custom metrics |
| Container | Container Insights (Azure Monitor Agent) | stdout/stderr logs, pod inventory, node metrics |
| Infrastructure | Managed Prometheus + Grafana | Kubernetes metrics, GPU metrics, custom Prometheus targets |

## Observability Best Practices for Government Workloads

**Use separate Application Insights resources per environment.** Create distinct resources for dev, staging, and production. Per-deployment onboarding makes this straightforward by pointing each deployment's `Instrumentation` CR at a different connection string.

**Restart deployments weekly.** The autoinstrumentation agent version is updated when pods restart. Regular restarts ensure you are running the latest version with the most recent security patches.

**Set up alerting early.** Use [Application Insights smart detection](https://learn.microsoft.com/en-us/azure/azure-monitor/alerts/alerts-overview) and Prometheus alert rules to catch anomalies before they impact citizens. Alert on response time degradation, failure rate spikes, and dependency failures.

**Control log volume with namespace filtering.** Use the `namespaceFilteringMode` setting in your Container Insights data collection configuration to limit log ingestion to namespaces you care about. This reduces costs and noise:

```json
{
  "interval": "1m",
  "namespaceFilteringMode": "Include",
  "namespaces": ["citizen-services", "payment-processing"],
  "enableContainerLogV2": true,
  "streams": ["Microsoft-ContainerLogV2"]
}
```

**Tag workloads with cloud role names.** If multiple services report to the same Application Insights resource, [set cloud role names](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-configuration#set-the-cloud-role-name-and-the-cloud-role-instance) so the Application Map accurately reflects your architecture.

## Why This Matters for Government

Government agencies running containerized workloads on AKS face unique pressures: strict uptime requirements for citizen-facing services, compliance mandates that require audit trails and visibility into application behavior, and lean IT teams that cannot afford to spend weeks manually instrumenting every microservice.

AKS autoinstrumentation directly addresses these challenges:

- **Faster time-to-value:** Government development teams can add full Application Insights monitoring to existing Java and Node.js deployments without modifying a single line of application code. This is critical for agencies that have inherited legacy codebases or rely on vendor-developed applications where source access may be limited.
- **Compliance-ready observability:** Distributed traces and correlated logs create an auditable record of request flows across services, supporting compliance requirements around system monitoring and incident response.
- **Reduced operational burden:** With autoinstrumentation, a single platform team can enable observability across dozens of microservices via namespace-wide onboarding, rather than coordinating code changes across multiple development teams.
- **Open standards alignment:** OpenTelemetry's vendor-neutral approach aligns with government procurement best practices that favor open standards and avoid vendor lock-in.

> **Azure Government note:** This preview is currently available in **Azure public cloud** only. Government customers using Azure commercial subscriptions can use it today. If your AKS workloads run in Azure Government regions, monitor the [Azure Government services availability page](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) for updates on when this feature will be supported. In the meantime, the [Azure Monitor OpenTelemetry Distro](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable) can be added manually to workloads running in Azure Government.

## Additional Resources

- [AKS Autoinstrumentation Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/app/kubernetes-codeless)
- [Azure Monitor OpenTelemetry Distro](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-enable)
- [Kubernetes Monitoring with Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-overview)
- [Enable Monitoring for AKS Clusters](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/kubernetes-monitoring-enable)
- [OpenTelemetry Project](https://opentelemetry.io/)
- [Azure Monitor Application Insights Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/app/opentelemetry-overview)

