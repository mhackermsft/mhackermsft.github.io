---
title: 'Azure Managed Grafana for Government Cloud: Hands-on Observability with KQL, Bicep, and Dashboard Patterns'
date: 2026-05-13T12:45:04+00:00
author: Mike Hacker
tags:
- Data Platform
- How To
- Security
categories:
- Azure Government
summary: Deploy Azure Managed Grafana in Azure Government for infrastructure observability with hands-on Bicep templates, KQL queries, and dashboard patterns built for government operations teams.
draft: false
image_prompt: A vast astronomical observatory dome interior at twilight, with a massive telescope pointed upward through the open slit toward a star-filled sky. Multiple smaller instruments and brass eyepieces are arranged around the base of the telescope on a rotating platform, all oriented in different directions but connected to the same central pillar. Warm amber light from wall-mounted fixtures contrasts with the cool blue of the night sky visible through the dome opening. Cables and conduits run along the curved walls connecting the instruments. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
audio: audio.mp3
---

Government IT teams have long struggled with fragmented monitoring tools: one console for VM metrics, another for log queries, a third for container health. Azure Managed Grafana brings these signals together in a single, fully managed visualization platform that is now generally available in Azure Government (Fairfax regions). Combined with the new Azure Monitor dashboards with Grafana experience embedded directly in the Azure portal, operations teams finally have a first-class, no-compromise observability stack that meets government compliance requirements.

This post walks through deploying Azure Managed Grafana in Azure Government with Bicep, writing KQL queries for infrastructure dashboards, and configuring role-based access patterns for multi-team government environments.

## What Azure Managed Grafana Brings to Government

Azure Managed Grafana is a fully managed service built on Grafana Labs software, operated and supported by Microsoft. It provides built-in integration with Azure Monitor, Azure Data Explorer, and Azure Managed Prometheus, all authenticated through Microsoft Entra ID and controlled by Azure RBAC.

In Azure Government, the Standard tier is available with the following key capabilities ([source](https://learn.microsoft.com/en-us/azure/managed-grafana/known-limitations#feature-availability-in-sovereign-clouds)):

| Feature | Azure Government Status |
|---|---|
| Private Link | Supported |
| Managed Private Endpoint | Supported |
| Team Sync with Entra ID | Preview |
| Zone Redundancy | Supported |
| Alerting | Supported |
| Deterministic Outbound IP | Supported |
| Enterprise Plugins | Not Supported |

For teams that need a zero-cost, zero-configuration starting point, **Azure Monitor dashboards with Grafana** delivers prebuilt Grafana dashboards directly in the Azure portal. This embedded experience supports Azure Monitor Metrics, Prometheus, Log Analytics (KQL), Application Insights traces, and Azure Resource Graph data sources at no additional cost ([source](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/grafana-plugin)).

## Deploying Azure Managed Grafana with Bicep

Infrastructure-as-code is non-negotiable for government environments where audit trails and repeatable deployments are required. The following Bicep template deploys an Azure Managed Grafana workspace in Azure Government with a system-assigned managed identity, zone redundancy, and an Azure Monitor workspace integration.

```bicep
@description('Name of the Grafana workspace')
param grafanaName string

@description('Azure region for deployment (e.g., usgovvirginia)')
param location string = resourceGroup().location

@description('Resource ID of the Azure Monitor workspace for Prometheus')
param azureMonitorWorkspaceId string = ''

@description('Enable zone redundancy for high availability')
param enableZoneRedundancy bool = true

resource grafana 'Microsoft.Dashboard/grafana@2024-10-01' = {
  name: grafanaName
  location: location
  sku: {
    name: 'Standard'
  }
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    apiKey: 'Enabled'
    deterministicOutboundIP: 'Enabled'
    zoneRedundancy: enableZoneRedundancy ? 'Enabled' : 'Disabled'
    publicNetworkAccess: 'Enabled'
    grafanaMajorVersion: '11'
    grafanaIntegrations: !empty(azureMonitorWorkspaceId) ? {
      azureMonitorWorkspaceIntegrations: [
        {
          azureMonitorWorkspaceResourceId: azureMonitorWorkspaceId
        }
      ]
    } : {}
    grafanaConfigurations: {
      users: {
        viewersCanEdit: false
        editorsCanAdmin: false
      }
    }
  }
}

// Grant the Grafana managed identity Monitoring Reader on the subscription
resource monitoringReaderRole 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(subscription().id, grafana.id, 'monitoring-reader')
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '43d0d8ad-25c7-4714-9337-8ba259a9fe05' // Monitoring Reader
    )
    principalId: grafana.identity.principalId
    principalType: 'ServicePrincipal'
  }
}

output grafanaEndpoint string = grafana.properties.endpoint
output grafanaPrincipalId string = grafana.identity.principalId
```

Deploy with the Azure CLI targeting Azure Government:

```bash
# Ensure you are logged into the Azure Government cloud
az cloud set --name AzureUSGovernment
az login

az deployment group create \
  --resource-group rg-monitoring \
  --template-file grafana.bicep \
  --parameters grafanaName='gov-ops-grafana' \
               azureMonitorWorkspaceId='/subscriptions/<sub-id>/resourceGroups/rg-monitoring/providers/Microsoft.Monitor/accounts/gov-prometheus'
```

> **Azure Government endpoint note:** The Grafana workspace endpoint will use the `.azure.us` domain rather than `.azure.com`. Ensure any firewall rules or DNS configurations account for this difference. Review the [Azure Government developer guide](https://learn.microsoft.com/en-us/azure/azure-government/documentation-government-developer-guide) for the full list of sovereign endpoint differences.

## KQL Queries for Infrastructure Observability

The Azure Monitor Logs data source in Grafana lets you run Kusto Query Language (KQL) queries directly against your Log Analytics workspaces. Below are production-ready queries designed for government infrastructure monitoring.

### VM Availability Heatmap

Track heartbeat gaps across your fleet to identify offline machines before users report issues:

```kql
Heartbeat
| where TimeGenerated > ago(24h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer, ResourceGroup
| extend MinutesSinceLastHeartbeat = datetime_diff('minute', now(), LastHeartbeat)
| extend Status = case(
    MinutesSinceLastHeartbeat <= 5, "Healthy",
    MinutesSinceLastHeartbeat <= 15, "Warning",
    "Critical"
  )
| project Computer, ResourceGroup, LastHeartbeat, MinutesSinceLastHeartbeat, Status
| order by MinutesSinceLastHeartbeat desc
```

### Disk Space Trending

Catch volumes approaching capacity before they cause application failures:

```kql
InsightsMetrics
| where TimeGenerated > ago(7d)
| where Namespace == "LogicalDisk" and Name == "FreeSpacePercentage"
| extend DiskPath = tostring(parse_json(Tags)["vm.azm.ms/mountId"])
| summarize AvgFreePercent = avg(Val) by bin(TimeGenerated, 1h), Computer, DiskPath
| where AvgFreePercent < 20
| order by AvgFreePercent asc
```

### Failed Login Attempts by Source

Security-relevant query that surfaces brute-force patterns:

```kql
SecurityEvent
| where TimeGenerated > ago(24h)
| where EventID == 4625 // Failed logon
| summarize FailedAttempts = count() by SourceIP = IpAddress, TargetAccount = TargetUserName, bin(TimeGenerated, 1h)
| where FailedAttempts > 10
| order by FailedAttempts desc
```

### Resource Provisioning Activity

Track infrastructure changes across your Azure Government subscriptions using Azure Activity logs:

```kql
AzureActivity
| where TimeGenerated > ago(7d)
| where OperationNameValue has "Microsoft.Compute" or OperationNameValue has "Microsoft.Network"
| where ActivityStatusValue == "Success"
| summarize OperationCount = count() by Caller, OperationNameValue, ResourceGroup, bin(TimeGenerated, 1d)
| order by OperationCount desc
```

In Grafana, each of these queries becomes a panel. Set the data source to **Azure Monitor**, select the target Log Analytics workspace, and paste the KQL into the query editor. Use Grafana variables (e.g., `$subscription`, `$resourceGroup`) to make dashboards dynamic across environments.

## Dashboard Configuration Patterns

### Pattern 1: Layered Operations Dashboard

Structure dashboards in three tiers that map to government operations workflows:

1. **Executive Row** - A top row with stat panels showing aggregate health: total VMs, healthy percentage, open critical alerts, and cost burn rate. Use the Azure Resource Graph data source to pull live resource counts.
2. **Investigation Row** - Time-series panels for CPU, memory, disk, and network metrics using Azure Monitor Metrics data source with `$resource` template variables.
3. **Detail Row** - Log panels with the KQL queries above, filtered by the same template variables so clicking a resource in the investigation row drills into its logs.

### Pattern 2: Multi-Agency Folder Structure

For shared Grafana instances serving multiple departments, organize dashboards into Grafana folders that align with Azure resource groups or management group hierarchy:

```
/Infrastructure-Operations/
  VM-Fleet-Health.json
  Network-Performance.json
  Storage-Capacity.json
/Security-Operations/
  Failed-Logins.json
  NSG-Flow-Analysis.json
/Application-Teams/
  Web-App-Performance.json
  API-Latency.json
```

Use Grafana folder-level permissions combined with Azure RBAC to ensure the security operations team sees only their dashboards, while infrastructure operators see theirs. Assign the **Grafana Viewer** role to read-only stakeholders and **Grafana Editor** to team members who build dashboards ([source](https://learn.microsoft.com/en-us/azure/managed-grafana/how-to-manage-access-permissions-users-identities)).

### Pattern 3: Prometheus Integration for Container Workloads

If your agency runs Azure Kubernetes Service, connect Azure Managed Prometheus to your Grafana instance. The Bicep template above already wires the integration. Then import the community Kubernetes dashboards:

1. In Grafana, navigate to **Dashboards** then **Import**.
2. Enter dashboard ID `3119` (Kubernetes cluster monitoring via Prometheus) or `15760` (Kubernetes views by pod).
3. Select the Prometheus data source pointed at your Azure Monitor workspace query endpoint.

The query endpoint for Azure Government follows the pattern `https://<workspace-name>.usgovvirginia.prometheus.monitor.azure.us` rather than the commercial `.azure.com` equivalent ([source](https://learn.microsoft.com/en-us/azure/azure-monitor/essentials/prometheus-grafana)).

## RBAC and Identity Configuration

Azure Managed Grafana uses four built-in roles that map to Entra ID assignments ([source](https://learn.microsoft.com/en-us/azure/managed-grafana/how-to-manage-access-permissions-users-identities)):

| Role | Purpose | Typical Assignment |
|---|---|---|
| Grafana Admin | Full control including data sources and RBAC | Platform team leads |
| Grafana Editor | Create and edit dashboards and alerts | Operations engineers |
| Grafana Viewer | Read-only dashboard access | Agency leadership, auditors |
| Grafana Limited Viewer | Home page only, no default permissions | Onboarding users before granular assignment |

Assign these roles through the standard Azure RBAC workflow on the Grafana resource's **Access Control (IAM)** blade. For the managed identity itself, grant **Monitoring Reader** on any subscription or resource group whose data the Grafana workspace should visualize. If using Prometheus, add **Monitoring Data Reader** on the Azure Monitor workspace.

```bash
# Grant Grafana Editor to a security group
az role assignment create \
  --assignee-object-id <entra-group-object-id> \
  --role "Grafana Editor" \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-monitoring/providers/Microsoft.Dashboard/grafana/gov-ops-grafana
```

## Why This Matters for Government

**Compliance-ready observability.** Azure Managed Grafana in Azure Government runs in FedRAMP High authorized regions with data residency guarantees. Private Link support means dashboard traffic never traverses the public internet, a hard requirement for many state and local agencies handling sensitive citizen data.

**Consolidated operational visibility.** Government agencies typically manage a mix of legacy VMs, modern containers, and PaaS services across multiple subscriptions. Grafana unifies metrics, logs, and traces from all of these into a single pane, reducing the number of tools operators need credentials for and simplifying incident response.

**Cost efficiency.** The Azure Monitor dashboards with Grafana experience is available at no additional cost in the Azure portal, giving agencies a powerful starting point. For teams that need alerting, private networking, and cross-organization sharing, the Standard tier of Azure Managed Grafana provides predictable per-user pricing without requiring self-hosted Grafana infrastructure.

**Identity-first security.** Every Grafana login flows through Microsoft Entra ID, the same identity provider government agencies already use. There are no separate Grafana credentials to manage, no additional identity silos, and all access is auditable through Azure Activity Logs.

**Infrastructure-as-code governance.** The Bicep template approach shown above means every Grafana deployment is version-controlled, peer-reviewed, and reproducible. This aligns with government change management requirements and makes it straightforward to replicate monitoring environments across development, staging, and production subscriptions.

## Getting Started

1. **Start with the portal experience.** Navigate to **Azure Monitor** then **Dashboards with Grafana** in the Azure Government portal to explore prebuilt dashboards for your existing resources at zero cost ([documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/visualize-use-grafana-dashboards)).
2. **Deploy Azure Managed Grafana** using the Bicep template above when you need alerting, private endpoints, or dedicated dashboards for your agency.
3. **Connect your data sources**: Azure Monitor (automatic), Prometheus (configure the query endpoint), and Azure Data Explorer (if applicable).
4. **Build dashboards iteratively** using the KQL patterns in this post, then use Grafana's **Save As** and folder structure to organize by team.
5. **Lock down access** with Entra ID role assignments matching your agency's organizational structure.

For the complete Azure Managed Grafana documentation, see the [official overview on Microsoft Learn](https://learn.microsoft.com/en-us/azure/managed-grafana/overview). For government-specific feature availability and endpoint differences, review the [Azure Government comparison guide](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure).
