---
title: 'Proving Application Resilience with Azure Chaos Studio: A Hands-On Fault Injection Guide for Government Workloads'
date: 2026-07-08T12:38:49+00:00
author: Mike Hacker
tags:
- How To
- App Modernization
categories:
- Azure
- Government
summary: 'A technical walkthrough of Azure Chaos Studio: how to design chaos experiments, enable targets through ARM REST with az rest, and build resilience validation into government disaster recovery and business continuity practices.'
draft: false
image_prompt: An Azure-themed resilience testing scene for state and local government workloads, showing controlled fault injection, monitored application health, and disaster recovery validation in a secure cloud operations center.
image: cover.png
audio: audio.mp3
---

Every disaster recovery plan looks bulletproof in a Word document. The uncomfortable truth is that many recovery time objectives (RTOs) and failover assumptions are not tested against real failure until a real outage forces the issue. For government IT teams responsible for 311 systems, permitting portals, public safety dispatch, and benefits eligibility platforms, saying an application probably fails over is not an acceptable answer to an auditor or a resident.

Chaos engineering flips that model. Instead of waiting for failure, you inject it deliberately, in a controlled way, and verify that redundancy, retries, monitoring, and failover behave the way the architecture diagram promises. [Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-overview) is the managed Azure service for that validation. It injects controlled disruptions such as virtual machine shutdowns, database failovers, DNS failures, and in-guest resource pressure.

## What Chaos Studio does

Chaos Studio has two authoring models to understand.

**Workspaces and Scenarios.** Microsoft Learn positions Workspaces as the fastest way to get started. A Workspace connects to a subscription, resource group, or service group scope, discovers resources, and recommends Scenario templates such as DNS Outage or Compute Zone Down. Each Scenario run produces a structured Scenario report that can be used for retrospectives, stakeholder communication, and compliance evidence. The Scenario documentation identifies Workspaces and Scenarios as public preview, so use your agency preview-governance process before treating them as a production control.

**Experiments.** Experiments are the classic model. They give you direct control over steps, branches, actions, targets, and selectors. Existing experiments remain supported. Experiments are also useful when you want the outage pattern stored as JSON in source control or when you need exact control over the target selector, fault duration, and parameters. Microsoft recommends starting new resilience testing with Workspaces and Scenarios when they fit the outage pattern, but this guide uses the experiment model because it is the most transparent way to see what Chaos Studio executes.

## Two kinds of faults

Chaos Studio supports two fault delivery mechanisms:

| Delivery mechanism | How it works | Examples |
|---|---|---|
| Service-direct | Runs against an Azure resource through Azure management APIs. No in-guest agent is required. | VM Shutdown, Virtual Machine Scale Set Shutdown, Cosmos DB Failover, Azure Cache for Redis reboot, NSG security-rule manipulation |
| Agent-based | Runs inside a VM or VM scale set instance through the Chaos Studio agent. | CPU pressure, memory pressure, process kill, network latency, network disconnect, packet loss |

From the [fault and action library](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-fault-library), these are especially relevant to government resilience testing:

| Target | Fault | Delivery | What it proves |
|---|---|---|---|
| Virtual Machine | VM Shutdown | Service-direct | Single-node loss and load balancer behavior |
| Virtual Machine Scale Set | Virtual Machine Scale Set Shutdown 2.0 | Service-direct | Zone-level failover for Uniform orchestration mode scale sets |
| AKS | AKS Chaos Mesh Pod Chaos | Service-direct | Pod failure and Kubernetes self-healing |
| Cosmos DB | Cosmos DB Failover | Service-direct | Single-write-account failover from the write region to a read region |
| VM or VMSS | Network Latency | Agent-based | Timeout tuning, retry policy behavior, and circuit-breaker logic |
| VM or VMSS | Network Disconnect | Agent-based | Dependency isolation and graceful degradation |

The zone-failover case maps directly to many disaster recovery conversations. The Virtual Machine Scale Set Shutdown 2.0 fault can target instances in specific availability zones through a selector filter. That is the safer way to answer: does this application survive losing a zone?

## Enabling a target and running a zone-shutdown experiment

A current operational detail matters: the Azure CLI reference includes an `az chaos` extension for Workspace and Scenario operations, while classic experiment authoring is still commonly driven through the ARM REST API. The examples below use `az rest` with the documented stable Chaos Studio REST API version, `2025-01-01`.

Chaos Studio cannot affect a resource until you explicitly onboard that resource as a target and enable the specific capability you plan to run. For a VM scale set zone-shutdown test, enable the VMSS target and the `Shutdown-2.0` capability. Set `VMSS_RESOURCE_ID` to the full ARM resource ID, including the leading `/subscriptions/...` path:

```bash
# Required inputs
SUB=<subscription-id>
RG=<resource-group-name>
NAME=<experiment-name>
VMSS_RESOURCE_ID=/subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/providers/Microsoft.Compute/virtualMachineScaleSets/<vmss-name>

# 1. Onboard the scale set as a Chaos Studio target
az rest --method put --url https://management.azure.com${VMSS_RESOURCE_ID}/providers/Microsoft.Chaos/targets/Microsoft-VirtualMachineScaleSet?api-version=2025-01-01 --body '{"properties":{}}'

# 2. Enable the zone-aware VMSS shutdown capability
az rest --method put --url https://management.azure.com${VMSS_RESOURCE_ID}/providers/Microsoft.Chaos/targets/Microsoft-VirtualMachineScaleSet/capabilities/Shutdown-2.0?api-version=2025-01-01 --body '{"properties":{}}'
```

Next, create an experiment definition. The key detail is that the availability zone belongs in the selector filter, not in the shutdown action parameters:

```json
{
  "location": "eastus",
  "identity": {
    "type": "SystemAssigned"
  },
  "properties": {
    "steps": [
      {
        "name": "Step1-DropZone",
        "branches": [
          {
            "name": "Branch1",
            "actions": [
              {
                "type": "continuous",
                "name": "urn:csci:microsoft:virtualMachineScaleSet:shutdown/2.0",
                "selectorId": "Zone1ScaleSet",
                "duration": "PT10M",
                "parameters": [
                  {
                    "key": "abruptShutdown",
                    "value": "false"
                  }
                ]
              }
            ]
          }
        ]
      }
    ],
    "selectors": [
      {
        "id": "Zone1ScaleSet",
        "type": "List",
        "targets": [
          {
            "type": "ChaosTarget",
            "id": "<vmss-resource-id>/providers/Microsoft.Chaos/targets/Microsoft-VirtualMachineScaleSet"
          }
        ],
        "filter": {
          "type": "Simple",
          "parameters": {
            "zones": [
              "1"
            ]
          }
        }
      }
    ]
  }
}
```

Save that file as `experiment-vmss-zone.json`, create the experiment, capture the system-assigned managed identity principal ID, grant the identity permission to operate on the target, and start the run:

```bash
EXPERIMENT_PRINCIPAL_ID=$(az rest --method put --url https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Chaos/experiments/$NAME?api-version=2025-01-01 --body @experiment-vmss-zone.json --query identity.principalId -o tsv)

az role assignment create --role 'Virtual Machine Contributor' --assignee-object-id $EXPERIMENT_PRINCIPAL_ID --assignee-principal-type ServicePrincipal --scope $VMSS_RESOURCE_ID

az rest --method post --url https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.Chaos/experiments/$NAME/start?api-version=2025-01-01
```

That RBAC assignment is intentionally scoped to the VM scale set resource. The Chaos Studio role-assignment guidance lists Virtual Machine Contributor as the suggested built-in role for service-direct VMSS faults. For stricter production controls, create a custom role containing only the actions required by the specific capability. Chaos Studio documentation notes that there are built-in roles for managing Chaos Studio operations, but not least-privileged built-in roles for every underlying resource fault action.

## From one-off test to continuous validation

The real value is not a single game day. It is catching resilience regressions before they reach production. Because the experiment definition is JSON and the run operation is REST-based, you can store the definition in source control and invoke the start call from Azure Pipelines, GitHub Actions, or a controlled runbook.

A practical maturity path looks like this:

1. Start in pre-production with a tightly scoped resource group or Workspace scope.
2. Run lower-risk agent-based pressure faults first, such as CPU or latency, after the agent is installed and outbound agent connectivity is approved.
3. Graduate to service-direct zone shutdowns once monitoring and incident communications are ready.
4. Correlate Chaos Studio results with Azure Monitor and application health checks to confirm the measured RTO.
5. Schedule recurring tests before election cycles, tax deadlines, open-enrollment windows, or other high-demand periods.

## Why This Matters for Government

Operational resilience is increasingly a governance requirement, not just an engineering preference. Microsoft documents Scenario reports as structured evidence for operational resilience testing, including what ran, which resources were targeted, what was skipped, and whether the run succeeded. For state and local agencies, that evidence can support continuity-of-operations documentation, disaster recovery testing records, and after-action reviews.

There is also a budget and trust dimension. Public-sector systems often carry years of accumulated assumptions about redundancy. Controlled fault injection turns those assumptions into verified facts or prioritized remediation items. That supports risk-based modernization decisions that are easier to defend in budget reviews, public records requests, and inspector general conversations.

Two caveats matter for government tenants. First, check regional availability before planning rollout. Chaos Studio has separate concepts for experiment deployment regions and target-resource regions, and the public region list is not the same thing as Azure Government availability. If workloads run in Azure Government, confirm service availability through the Products available by region page and the Azure Government documentation rather than assuming parity with commercial Azure. Second, blast radius is a governance decision. Use scoped targets, managed identities, custom roles where needed, pre-production-first sequencing, and change-management approvals before running faults against mission-critical systems.

## Getting started

Start with a non-production resource group. If your organization permits public preview features, create a Workspace, run a recommended Scenario, and review the Scenario report. Then reproduce one critical pattern as a classic experiment in JSON so your team understands the steps, branches, actions, selectors, capabilities, and RBAC that make the test work. Once the pattern is reviewed and automated, resilience moves from an assumption to an operating practice.

**Official references:**

- [What is Azure Chaos Studio?](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-overview)
- [Azure Chaos Studio fault and action library](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-fault-library)
- [Targets - Create Or Update REST API](https://learn.microsoft.com/en-us/rest/api/chaosstudio/targets/create-or-update)
- [Capabilities - Create Or Update REST API](https://learn.microsoft.com/en-us/rest/api/chaosstudio/capabilities/create-or-update)
- [Experiments - Create Or Update REST API](https://learn.microsoft.com/en-us/rest/api/chaosstudio/experiments/create-or-update)
- [Experiments - Start REST API](https://learn.microsoft.com/en-us/rest/api/chaosstudio/experiments/start)
- [Supported resource types and role assignments for Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-fault-providers)
- [Scenarios in Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-scenarios)
- [Scenario reports in Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-scenario-reports)
- [Regional availability of Azure Chaos Studio](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-region-availability)
- [Azure Chaos Studio limitations and known issues](https://learn.microsoft.com/en-us/azure/chaos-studio/chaos-studio-limitations)

