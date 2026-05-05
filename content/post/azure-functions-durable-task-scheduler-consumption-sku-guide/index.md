---
title: 'Azure Functions Durable Task Scheduler Consumption SKU: Build Pay-Per-Use Workflows and AI Agent Orchestrations'
date: 2026-05-05T12:11:47+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- How To
categories:
- Azure
summary: A hands-on technical guide to deploying durable workflows and AI agent orchestrations with the Durable Task Scheduler Consumption SKU, including Bicep templates, .NET/Python code examples, and cost optimization patterns for government workloads.
draft: false
image_prompt: A vast workshop table covered in hundreds of delicate hand-carved wooden puzzle pieces, each interlocking piece a different shape and wood grain, some assembled into completed sections showing a flowing chain pattern while others wait in organized rows. Warm amber light from a single overhead industrial lamp casts deep shadows between the pieces. Wood shavings and fine dust catch the light. A craftsman's hands are visible at the edge, carefully fitting one piece into the next in the chain. Shallow depth of field with rich bokeh in the background. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
audio: audio.mp3
---

Government IT teams building workflow automation and AI agent orchestrations now have a compelling pay-per-use option: the **Durable Task Scheduler Consumption SKU** for Azure Functions. This fully managed backend eliminates the need to provision storage accounts, manage queue infrastructure, or predict capacity upfront. You pay only for actions dispatched, making it ideal for variable workloads common in government environments: seasonal permit processing, intermittent citizen service requests, and bursty AI inference pipelines.

This guide walks through architecture patterns, deployment templates, and working code examples to get your team building production durable workflows on the Consumption SKU.

## What Is the Durable Task Scheduler?

The [Durable Task Scheduler](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/durable-task-scheduler) is Azure's recommended storage backend for Durable Functions. Unlike the legacy Azure Storage provider (where you manage your own queues and tables), the Durable Task Scheduler is a purpose-built, fully managed backend-as-a-service. Your apps connect via gRPC, and the scheduler handles all state persistence, message dispatching, and scaling internally.

The scheduler offers two billing SKUs:

| Feature | Dedicated SKU | Consumption SKU |
|---------|--------------|------------------|
| **Billing model** | Fixed monthly per Capacity Unit | Pay per action dispatched |
| **Max throughput** | 2,000 actions/sec per CU | 500 actions/sec |
| **Data retention** | Up to 90 days | Up to 30 days |
| **High availability** | Supported (3 CUs) | Not available |
| **Best for** | Production at scale | Dev/test, variable workloads |

An *action* is any message dispatched to your application: starting an orchestration, scheduling an activity, completing a timer, processing an external event, or returning activity results. A simple three-activity orchestration incurs approximately 7 actions (1 orchestrator start + 2 per activity for scheduling and result processing).

## Architecture: How It Fits Together

The deployment architecture separates compute from state management:

1. **Azure Functions app** (Flex Consumption or Consumption plan) provides serverless compute
2. **Durable Task Scheduler** (Consumption SKU) manages all orchestration state
3. **User-Assigned Managed Identity** authenticates the function app to the scheduler
4. **Task Hub** is a logical container for your orchestration instances within the scheduler

Your function app connects to the scheduler endpoint (e.g., `myscheduler.eastus.durabletask.io`) over TLS-secured gRPC. Work items are pushed to your app, eliminating polling overhead.

## Deploying with Bicep

Here is a complete Bicep template that provisions a Durable Task Scheduler with the Consumption SKU, a task hub, a user-assigned managed identity, and the required role assignment:

```bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Name of the Durable Task Scheduler')
param schedulerName string = 'dts-workflows-consumption'

@description('Name of the task hub')
param taskHubName string = 'citizenservices'

@description('Name of the user-assigned managed identity')
param identityName string = 'id-durable-task-worker'

// Durable Task Data Contributor role definition ID
var durableTaskDataContributorRoleId = subscriptionResourceId(
  'Microsoft.Authorization/roleDefinitions',
  '58da3a78-d39e-4f1a-b8d0-3a7a1ef093b4'
)

resource managedIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: identityName
  location: location
}

resource scheduler 'Microsoft.DurableTask/schedulers@2026-02-01' = {
  name: schedulerName
  location: location
  properties: {
    ipAllowlist: [
      '0.0.0.0/0'  // Restrict to your VNet ranges in production
    ]
    sku: {
      name: 'Consumption'
    }
  }
}

resource taskHub 'Microsoft.DurableTask/schedulers/taskHubs@2026-02-01' = {
  parent: scheduler
  name: taskHubName
  properties: {}
}

resource roleAssignment 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(taskHub.id, managedIdentity.id, durableTaskDataContributorRoleId)
  scope: taskHub
  properties: {
    roleDefinitionId: durableTaskDataContributorRoleId
    principalId: managedIdentity.properties.principalId
    principalType: 'ServicePrincipal'
  }
}

output schedulerEndpoint string = scheduler.properties.endpoint
output taskHubName string = taskHub.name
output identityClientId string = managedIdentity.properties.clientId
```

Deploy with the Azure CLI:

```bash
az group create --name rg-durable-workflows --location eastus
az deployment group create \
  --resource-group rg-durable-workflows \
  --template-file main.bicep
```

> **Production note:** Replace the `0.0.0.0/0` IP allowlist with your function app's outbound IP ranges or use [private endpoints](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/durable-task-scheduler-private-endpoints) for VNet-integrated deployments.

## Configuring Your Function App

After deploying infrastructure, configure your function app to connect to the scheduler.

**host.json:**
```json
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle",
    "version": "[4.32.0, 5.0.0)"
  },
  "extensions": {
    "durableTask": {
      "hubName": "%TASKHUB_NAME%",
      "storageProvider": {
        "type": "azureManaged",
        "connectionStringName": "DURABLE_TASK_SCHEDULER_CONNECTION_STRING"
      }
    }
  }
}
```

**Application settings (set via Azure CLI or Bicep):**
```bash
az functionapp config appsettings set \
  --name my-function-app \
  --resource-group rg-durable-workflows \
  --settings \
    TASKHUB_NAME="citizenservices" \
    DURABLE_TASK_SCHEDULER_CONNECTION_STRING="Endpoint=https://dts-workflows-consumption.eastus.durabletask.io;Authentication=ManagedIdentity;ClientID=<your-identity-client-id>"
```

## .NET Example: Multi-Step Permit Processing Workflow

Here is a real-world orchestration pattern: a citizen permit application that fans out approval checks in parallel, then chains a final notification step.

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask;
using Microsoft.DurableTask.Client;

public static class PermitProcessingOrchestration
{
    [Function(nameof(ProcessPermitApplication))]
    public static async Task<PermitResult> ProcessPermitApplication(
        [OrchestrationTrigger] TaskOrchestrationContext context)
    {
        var application = context.GetInput<PermitApplication>();

        // Step 1: Validate application data
        var validation = await context.CallActivityAsync<ValidationResult>(
            nameof(ValidateApplication), application);

        if (!validation.IsValid)
            return new PermitResult { Status = "Rejected", Reason = validation.Error };

        // Step 2: Fan-out parallel compliance checks
        var complianceTasks = new List<Task<ComplianceResult>>
        {
            context.CallActivityAsync<ComplianceResult>(nameof(CheckZoning), application),
            context.CallActivityAsync<ComplianceResult>(nameof(CheckEnvironmental), application),
            context.CallActivityAsync<ComplianceResult>(nameof(CheckBuildingCode), application)
        };

        var results = await Task.WhenAll(complianceTasks);

        // Step 3: Aggregate results and notify
        if (results.All(r => r.Passed))
        {
            await context.CallActivityAsync(nameof(IssuePermit), application);
            await context.CallActivityAsync(nameof(NotifyCitizen), 
                new Notification { Email = application.Email, Status = "Approved" });
            return new PermitResult { Status = "Approved" };
        }

        var failures = results.Where(r => !r.Passed).Select(r => r.Reason);
        await context.CallActivityAsync(nameof(NotifyCitizen),
            new Notification { Email = application.Email, Status = "Denied", Details = failures });

        return new PermitResult { Status = "Denied", Reason = string.Join("; ", failures) };
    }

    [Function(nameof(ValidateApplication))]
    public static ValidationResult ValidateApplication(
        [ActivityTrigger] PermitApplication app)
    {
        // Validate required fields, check for duplicates, etc.
        if (string.IsNullOrEmpty(app.ParcelId))
            return new ValidationResult { IsValid = false, Error = "Missing parcel ID" };
        return new ValidationResult { IsValid = true };
    }

    [Function(nameof(CheckZoning))]
    public static async Task<ComplianceResult> CheckZoning(
        [ActivityTrigger] PermitApplication app)
    {
        // Call external zoning API
        return new ComplianceResult { Passed = true, CheckName = "Zoning" };
    }

    // Additional activity functions omitted for brevity...
}
```

This orchestration consumes approximately 11 actions on the Consumption SKU (1 start + 2 validation + 6 parallel checks + 2 notification). At Consumption pricing, even 100,000 monthly permit applications remain extremely cost-effective.

## Python Example: AI Agent Orchestration

Durable Functions excels at coordinating multi-step AI agent workflows where each step calls a different model or service. Here is a Python orchestration that implements a research-summarize-review agent pattern:

```python
import azure.functions as func
import azure.durable_functions as df

def orchestrator_function(context: df.DurableOrchestrationContext):
    """AI research agent: gather, summarize, then quality-check."""
    query = context.get_input()

    # Step 1: Research agent gathers relevant documents
    documents = yield context.call_activity("GatherDocuments", query)

    # Step 2: Fan-out - summarize each document in parallel
    summary_tasks = [
        context.call_activity("SummarizeDocument", doc)
        for doc in documents
    ]
    summaries = yield context.task_all(summary_tasks)

    # Step 3: Synthesize all summaries into a final report
    report = yield context.call_activity("SynthesizeReport", {
        "query": query,
        "summaries": summaries
    })

    # Step 4: Quality review agent checks for accuracy
    review = yield context.call_activity("QualityReview", report)

    if not review["approved"]:
        # Retry with feedback loop
        report = yield context.call_activity("ReviseReport", {
            "report": report,
            "feedback": review["feedback"]
        })

    return report

main = df.Orchestrator.create(orchestrator_function)
```

Each activity function can call Azure OpenAI Service independently, and the orchestrator maintains state across all LLM calls without you managing any queues or databases.

## Cost Optimization Patterns for Government

The Consumption SKU pricing model aligns perfectly with government budgeting:

**1. Action-based billing eliminates waste.** Seasonal workflows (tax filing, enrollment periods, annual reporting) incur zero scheduler cost when idle. You only pay when orchestrations actually run.

**2. Share schedulers across teams.** A single Consumption SKU scheduler supports up to 5 task hubs. Multiple department applications can share infrastructure while maintaining logical isolation.

**3. Right-size your compute layer.** Pair the Consumption SKU scheduler with Azure Functions Flex Consumption plan for true double-serverless: both your compute and your orchestration state scale to zero.

**4. Use the emulator for development.** The [Docker-based emulator](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/develop-with-durable-task-scheduler#durable-task-scheduler-emulator) runs locally with zero cloud cost:

```bash
docker pull mcr.microsoft.com/dts/dts-emulator:latest
docker run -d -p 8080:8080 -p 8082:8082 mcr.microsoft.com/dts/dts-emulator:latest
```

Developers test their full orchestration logic locally before deploying to Azure.

## Why This Matters for Government

**Budget predictability with zero idle cost.** Government procurement cycles demand predictable spending. The Consumption SKU charges nothing when workflows are inactive, eliminating the need to justify fixed infrastructure costs for intermittent workloads like seasonal licensing, election systems, or periodic compliance reporting.

**Managed identity-only authentication.** The Durable Task Scheduler does not support connection strings with storage keys. It requires managed identity (RBAC) authentication exclusively. This aligns with Zero Trust mandates and simplifies security audits by eliminating shared secrets from your deployment.

**AI agent orchestration without infrastructure complexity.** As government agencies adopt Azure OpenAI for citizen services, document processing, and decision support, the Durable Task Scheduler provides fault-tolerant coordination of multi-model AI pipelines without requiring Kubernetes, custom message brokers, or dedicated VMs.

**Compliance-friendly architecture.** The scheduler is available in Azure commercial regions. For organizations using Azure Government, continue monitoring [regional availability](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/durable-task-scheduler#supported-regions) as the service expands.

**Lower operational burden.** With the legacy Azure Storage provider, teams manage table storage, queue configurations, and partition scaling themselves. The Durable Task Scheduler eliminates this operational overhead entirely, freeing IT staff to focus on application logic rather than infrastructure.

## Getting Started Checklist

1. Install the Azure CLI Durable Task extension: `az extension add --name durabletask`
2. Deploy the Bicep template above to create your scheduler and task hub
3. Run the Docker emulator locally for development
4. Update your `host.json` to use the `azureManaged` storage provider
5. Assign the `Durable Task Data Contributor` role to your managed identity
6. Deploy your function app and monitor via the [built-in dashboard](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/durable-task-scheduler-dashboard)

## Further Reading

- [Durable Task Scheduler overview](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/durable-task-scheduler)
- [Durable Task Scheduler billing](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/durable-task-scheduler-billing)
- [Quickstart: Configure Durable Functions with Durable Task Scheduler](https://learn.microsoft.com/en-us/azure/durable-task/scheduler/quickstart-durable-task-scheduler)
- [Bicep reference: Microsoft.DurableTask/schedulers](https://learn.microsoft.com/en-us/azure/templates/microsoft.durabletask/schedulers)
- [Durable Functions patterns and orchestrations](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-orchestrations)
- [Durable Task Scheduler pricing](https://azure.microsoft.com/pricing/details/durable-task-scheduler/)

