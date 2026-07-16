---
title: 'Long-Running MCP Tools on Azure Functions: A Durable Functions Deep Dive'
date: 2026-07-16T16:19:30+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- How To
categories:
- Azure
- AI
summary: How government developers can build resilient AI agent tool backends by pairing the Azure Functions MCP extension with Durable Functions for long-running asynchronous work.
draft: false
image_prompt: 'A bright workshop scene with a long line of oversized paper job folders moving through a sturdy mechanical relay track, one folder freshly stamped and handed off into a clearly labeled waiting tray shape without any readable markings, while a central brass-and-glass mechanism keeps the folders in order under warm cinematic light. The composition emphasizes asynchronous long-running work: one request is quickly handed off while the durable machine continues processing the backlog in the background. No text, letters, numbers, or writing anywhere in the image.'
image: cover.png
audio: audio.mp3
---

AI agents are only as useful as the tools they can call. When an agent asks a tool to "reprocess last quarter's permit backlog" or "run the eligibility check across every record," that work can take minutes or hours. A synchronous MCP tool call is the wrong shape for that work: HTTP responses can time out, serverless workers can restart, and in-memory progress can disappear.

This pattern pairs the Azure Functions MCP extension with Durable Functions. The MCP tool returns a job ID quickly, Durable Functions runs the long workflow with durable state, and the agent polls a second tool for status until the result is ready.

## The timeout problem with synchronous MCP tools

The [Azure Functions MCP extension](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-mcp) lets an Azure Functions app expose remote MCP tools. Clients connect over Streamable HTTP to `/runtime/webhooks/mcp`, discover tools, and invoke them.

That works well for fast tools, but two platform realities matter for long-running work:

- Azure Functions HTTP-triggered responses have a **230-second ceiling** because of the Azure Load Balancer idle timeout. The [Azure Functions hosting options](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale) documentation says this limit applies regardless of the function app timeout setting.
- A serverless worker can scale in or be replaced. If progress exists only in process memory, the model has no durable record to query after a restart.

For an agent, an ambiguous timeout is dangerous. It cannot tell whether the operation failed, succeeded, or is still running. The safer shape is asynchronous request-reply: return a tracking handle, run the work durably, and let the agent poll for status.

## Durable Functions provides the state machine

[Durable Functions](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview) is an Azure Functions extension for stateful workflows. The runtime manages state, checkpoints, retries, and recovery for long-running orchestrations.

The core pieces are straightforward:

- **Orchestrator functions** define workflow control flow. They must be deterministic and should not perform direct I/O.
- **Activity functions** perform I/O and compute. Completed activity results are recorded in orchestration history.
- **Entity functions** are optional and hold small pieces of long-lived state.

Durable Functions also documents the polling-consumer pattern. With an HTTP-triggered starter, `CreateCheckStatusResponse` can return a 202 response with a `statusQueryGetUri`. For an MCP-only interface, a custom status tool is usually cleaner because it hides Durable's full management payload and gives the model only the fields it needs.

## The architecture

Use three functions in one function app:

1. A **start tool** schedules the orchestration and returns a job ID.
2. A **status tool** queries orchestration metadata and returns model-friendly JSON.
3. The **orchestrator and activities** run the long, resumable workflow.

The agent calls `start_reprocessing`, receives `jobId`, and then calls `check_job` until the status is `Completed`, `Failed`, `Terminated`, or another terminal state.

### Start tool

```csharp
using System.Text.Json;
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Extensions.Mcp;
using Microsoft.DurableTask.Client;

public class ReprocessTools
{
    [Function(nameof(StartReprocess))]
    public async Task<string> StartReprocess(
        [McpToolTrigger("start_reprocessing",
            "Starts a long-running batch reprocessing job and returns a job id to poll.")]
            ToolInvocationContext context,
        [McpToolProperty("quarter", "Fiscal quarter to reprocess, e.g. 2026-Q1", isRequired: true)]
            string quarter,
        [DurableClient] DurableTaskClient client)
    {
        string instanceId = await client.ScheduleNewOrchestrationInstanceAsync(
            nameof(ReprocessOrchestrator), quarter);

        return JsonSerializer.Serialize(new
        {
            jobId = instanceId,
            status = "Running",
            message = $"Reprocessing {quarter} started. Poll check_job with this jobId."
        });
    }
}
```

The MCP attributes come from `Microsoft.Azure.Functions.Worker.Extensions.Mcp`. The Durable client binding for .NET isolated apps comes from `Microsoft.Azure.Functions.Worker.Extensions.DurableTask`.

### Orchestrator and activities

```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.DurableTask;

[Function(nameof(ReprocessOrchestrator))]
public static async Task<int> ReprocessOrchestrator(
    [OrchestrationTrigger] TaskOrchestrationContext context)
{
    string quarter = context.GetInput<string>()!;

    string[] batches = await context.CallActivityAsync<string[]>(
        nameof(GetBatches), quarter);

    var tasks = new List<Task<int>>();
    foreach (string batch in batches)
    {
        tasks.Add(context.CallActivityAsync<int>(nameof(ProcessBatch), batch));
    }

    int[] results = await Task.WhenAll(tasks);
    return results.Sum();
}

[Function(nameof(GetBatches))]
public static Task<string[]> GetBatches([ActivityTrigger] string quarter)
{
    return Task.FromResult(new[] { $"{quarter}-batch-001", $"{quarter}-batch-002" });
}

[Function(nameof(ProcessBatch))]
public static async Task<int> ProcessBatch([ActivityTrigger] string batchId)
{
    await Task.Delay(TimeSpan.FromSeconds(1));
    return 100;
}
```

This is the Durable Functions fan-out/fan-in pattern: start independent activity calls, wait for all of them, and aggregate the result. Completed activity outputs are stored in orchestration history, so orchestration replay does not rerun completed activities.

### Status tool

```csharp
[Function(nameof(CheckJob))]
public async Task<string> CheckJob(
    [McpToolTrigger("check_job", "Checks the status of a reprocessing job by id.")]
        ToolInvocationContext context,
    [McpToolProperty("jobId", "The job id returned by start_reprocessing", isRequired: true)]
        string jobId,
    [DurableClient] DurableTaskClient client)
{
    OrchestrationMetadata? meta =
        await client.GetInstancesAsync(jobId, getInputsAndOutputs: true);

    if (meta is null)
        return JsonSerializer.Serialize(new { jobId, status = "NotFound" });

    return JsonSerializer.Serialize(new
    {
        jobId,
        status = meta.RuntimeStatus.ToString(),
        result = meta.RuntimeStatus == OrchestrationRuntimeStatus.Completed
            ? meta.ReadOutputAs<int>()
            : (int?)null
    });
}
```

The method name `GetInstancesAsync` is plural even when querying one instance ID. Request inputs and outputs only when the status response needs them, because the Durable client documentation calls out the extra bandwidth, serialization, and memory cost.

## Hosting on Flex Consumption

For new serverless function apps, the [hosting options](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale) documentation recommends the **Flex Consumption plan**, which is generally available. Flex Consumption supports event-driven scale-out, virtual network integration, configurable memory sizes, and optional always-ready instances. Its documented maximum scale-out is up to 1,000 instances, and Durable Functions triggers scale together as a group.

Configuration essentials:

- Use extension bundle version `[4.0.0, 5.0.0)` in `host.json` when you rely on extension bundles.
- For C#, the MCP extension supports only the isolated worker model.
- Use Streamable HTTP unless a client specifically requires Server-Sent Events. The MCP docs note that newer protocol versions deprecate SSE.
- If you use SSE, the MCP extension relies on Azure Queue storage in the host storage account. With identity-based connections, grant at least **Storage Queue Data Contributor** and **Storage Queue Data Message Processor**.
- Hosted MCP endpoints require the `mcp_extension` system key in the `x-functions-key` header unless you configure anonymous webhook authorization and provide another authorization layer.

Retrieve the MCP system key with:

```bash
az functionapp keys list --resource-group <RESOURCE_GROUP> --name <APP_NAME>   --query systemKeys.mcp_extension --output tsv
```

Durable Functions supports multiple storage providers. The current storage-provider documentation identifies **Durable Task Scheduler** as the recommended backend for Durable Functions, while Azure Storage remains available as a bring-your-own storage provider.

## Why This Matters for Government

State and local agencies are moving from AI pilots to production agents that touch systems of record: eligibility, permitting, records requests, public health workflows, and case management. Those operations are rarely instantaneous, and they are rarely allowed to fail silently.

- **Reliability and auditability.** Durable orchestration history gives teams a clearer record of what ran, when it ran, and whether it completed, failed, or was terminated. It does not replace agency logging or records retention, but it is much stronger than an in-memory request.
- **Compliance-aware design.** Microsoft publishes Azure guidance for FedRAMP, CJIS, HIPAA, and IRS Publication 1075. Those offerings do not make an app compliant by themselves, but they help teams plan identity, encryption, logging, networking, and operational controls.
- **Cost discipline.** Flex Consumption on-demand instances can scale to zero when there is no work. If you configure always-ready instances to reduce cold starts, those baseline instances are billed, so use them deliberately.
- **Cloud choice.** Azure Government and global Azure use the same underlying technologies, but service availability and feature configuration can differ by cloud and region. Confirm Flex Consumption, Durable Functions, Durable Task Scheduler, storage, and networking availability before committing a production design.
- **Security boundary.** The model receives a small MCP surface: start a job and check a job. The heavy work can run behind virtual network integration, managed identity, storage RBAC, and application-specific authorization.

## Getting started

Add `Microsoft.Azure.Functions.Worker.Extensions.Mcp` and `Microsoft.Azure.Functions.Worker.Extensions.DurableTask` to a C# isolated-worker app. Build a start tool, an orchestrator, activity functions, and a status tool. Then deploy to Flex Consumption or another Functions hosting option that fits your operational requirements.

For details, read the [MCP bindings overview](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-mcp), the [MCP tool trigger reference](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-mcp-tool-trigger), the [Durable Functions overview](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview), and the [HTTP features in Durable Functions](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-http-features). Together they give your agents tools that can take their time without timing out the request that started them.

