---
title: Building MCP Servers in C# to Ground AI Coding Agents in Real Docs and Live Apps
date: 2026-08-28T13:44:52+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
categories:
- AI
- App Modernization
summary: 'A hands-on guide for government .NET teams to build two C# Model Context Protocol servers: one that grounds AI coding agents in authoritative documentation, and one that inspects and drives a running app through narrow, auditable tools.'
draft: false
image_prompt: 'A secure Azure-themed architecture illustration showing two C# MCP servers: one read-only documentation grounding server connected to approved runbooks, and one controlled live-app inspection server connected to a running app, with GitHub Copilot as the client and government audit controls around the system.'
image: cover.png
audio: audio.mp3
---

AI coding agents are only as good as the context they can reach. Ask an agent to fix a bug in an internal line-of-business app and it can still propose an API that does not exist, cite a framework version your team no longer uses, or miss behavior that only appears at runtime. The issue is grounding. The agent needs a reliable way to read authoritative documentation and, when appropriate, inspect the actual application while it is running.

The [Model Context Protocol (MCP)](https://learn.microsoft.com/en-us/dotnet/ai/get-started-mcp) is an open protocol for connecting AI applications to external tools and data sources. MCP uses a client-server architecture: a host such as GitHub Copilot in Visual Studio Code, GitHub Copilot agent mode, Visual Studio, or Microsoft Agent Framework connects through MCP clients to one or more servers. Those servers can expose tools, resources, and prompts that you control.

For government .NET teams, a useful pattern is to split that work into two servers: a read-only grounding server for approved documentation, and a live-app server for controlled inspection and interaction with a running application.

## Why two servers, not one

The two jobs are different, and separating them keeps each server smaller, easier to test, and easier to deploy under different controls.

**Server one: the grounding server.** Its job is read-only knowledge. It exposes approved API references, runbooks, coding standards, and architecture patterns so the agent can retrieve grounded answers instead of guessing. In MCP terms, this maps naturally to resources, which are addressable pieces of context, plus a search tool that returns relevant passages.

**Server two: the live-app server.** Its job is observation and narrowly scoped action against a running process. It exposes tools that let the agent query the current visual tree of an app, read a control's state, or invoke a specific command. The agent can reason about what the app actually looks like at runtime instead of relying only on source code.

The official [MCP C# SDK](https://www.nuget.org/packages/ModelContextProtocol) is distributed on NuGet and maintained through collaboration between Microsoft, Anthropic, and the MCP open protocol organization. As of August 28, 2026, the current stable NuGet package is version 2.2.0, published on August 13, 2026. The [Microsoft MCP server project template](https://learn.microsoft.com/en-us/dotnet/ai/quickstarts/build-mcp-server) is still marked preview in the Microsoft Learn quickstart, so pin package versions and review release notes before adopting new template features in production.

## Scaffolding a server

With the .NET 10 SDK, you can generate a server from the Microsoft MCP server project template. The Microsoft Learn quickstart shows this flow:

```bash
dotnet new install Microsoft.McpServer.ProjectTemplates
dotnet new mcpserver -n GovDocsMcpServer
```

You can also wire the server by hand in a console app when you want direct control over hosting and dependency injection:

```bash
dotnet new console -n GovDocsMcpServer
cd GovDocsMcpServer
dotnet add package ModelContextProtocol --version 2.2.0
dotnet add package Microsoft.Extensions.Hosting --version 10.0.11
```

The basic `Program.cs` registers the server, selects the stdio transport, and scans the assembly for tools:

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;
using ModelContextProtocol.Server;

var builder = Host.CreateApplicationBuilder(args);

// MCP over stdio communicates on stdin/stdout, so route logs to stderr.
builder.Logging.AddConsole(o =>
    o.LogToStandardErrorThreshold = LogLevel.Trace);

builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();

await builder.Build().RunAsync();
```

The stdio transport is useful for sensitive development environments because the MCP server runs as a local child process of the editor and communicates over standard input and standard output. That can reduce the server's network exposure, but it does not remove the need to configure the AI host according to agency data-handling policy. Any context returned to the host must still be governed by the controls that apply to that host and model.

## The grounding server: docs as resources and a search tool

Start with a service that loads approved documentation from an internal repository, document store, or curated set of Markdown runbooks. Register it with dependency injection so tools can consume it:

```csharp
builder.Services.AddSingleton<DocsService>();
```

Then expose retrieval as a tool. Tools are methods decorated with MCP SDK attributes. The SDK reads `Description` values and exposes that metadata to the client, so treat descriptions as part of the product experience.

```csharp
using System.ComponentModel;
using System.Text.Json;
using ModelContextProtocol.Server;

[McpServerToolType]
public static class DocsTools
{
    [McpServerTool, Description(
        "Search the agency's approved engineering documentation and return " +
        "the most relevant passages with their source paths.")]
    public static async Task<string> SearchDocs(
        DocsService docs,
        [Description("A natural-language query describing what to find")]
        string query,
        [Description("Maximum passages to return")]
        int maxResults = 5)
    {
        var hits = await docs.Search(query, maxResults);
        return JsonSerializer.Serialize(hits);
    }
}
```

Returning the source path with every passage gives reviewers a direct path back to approved guidance. For higher-quality retrieval, back `DocsService.Search` with a vector or full-text index that your agency already governs. Where documentation is a set of addressable files, expose those files as MCP resources too. In the MCP resource model, clients discover resources with `resources/list` and read them with `resources/read`, which lets an agent pull a whole runbook into context on demand.

## The live-app server: inspect and drive a running app

The second server can run in-process with the app or connect through a controlled inspection interface. Keep every action narrow and intentional:

```csharp
using System.ComponentModel;
using System.Text.Json;
using ModelContextProtocol.Server;

[McpServerToolType]
public sealed class LiveAppTools
{
    private readonly IAppInspector _app;

    public LiveAppTools(IAppInspector app) => _app = app;

    [McpServerTool, Description(
        "Return the current visual tree of the running app as JSON.")]
    public async Task<string> GetVisualTree()
        => JsonSerializer.Serialize(await _app.CaptureVisualTreeAsync());

    [McpServerTool, Description(
        "Read the current value and state of a control by automation id.")]
    public async Task<string> GetControlState(
        [Description("The automation id of the target control")] string automationId)
        => JsonSerializer.Serialize(await _app.GetControlStateAsync(automationId));

    [McpServerTool, Description(
        "Invoke a named, allow-listed command on the running app.")]
    public async Task<string> InvokeCommand(
        [Description("The allow-listed command or control to invoke")] string target)
        => JsonSerializer.Serialize(await _app.InvokeAsync(target));
}
```

The agent can capture current state, see that a control is collapsed at runtime, propose a change, invoke a specific allow-listed command, and capture state again. The server, not the model, decides which observations and actions exist.

Treat the live-app server as a privileged automation surface. Do not expose a general command runner. Return structured JSON rather than screen text so clients can parse results reliably. The MCP specification recommends a human in the loop for tool use, and GitHub Copilot workflows prompt users when tools run, but approval prompts are not a substitute for server-side controls. Validate inputs, implement access controls, rate-limit calls where appropriate, log tool use, and keep sensitive operations behind explicit allow lists.

## Going remote for team-wide reuse

A local stdio server is a good starting point for one developer. A grounding server built on shared agency documentation may also make sense as a shared service. For ASP.NET Core HTTP hosting with the current [ModelContextProtocol.AspNetCore](https://www.nuget.org/packages/ModelContextProtocol.AspNetCore) package, add the HTTP package and use `WithHttpTransport()` with `MapMcp()`:

```bash
dotnet new web -n GovDocsMcpHttpServer
cd GovDocsMcpHttpServer
dotnet add package ModelContextProtocol.AspNetCore --version 2.2.0
```

```csharp
using ModelContextProtocol.AspNetCore;
using ModelContextProtocol.Server;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddMcpServer()
    .WithHttpTransport(options =>
    {
        options.SessionMode = HttpServerSessionMode.Stateless;
    })
    .WithToolsFromAssembly();

var app = builder.Build();
app.MapMcp();
await app.RunAsync();
```

For HTTP-hosted MCP servers, configure authentication, restrict allowed host names, validate incoming origins, and avoid binding local development servers to all network interfaces unless you have a specific reason and compensating controls.

You can also host remote MCP tools with the [Model Context Protocol bindings for Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-mcp). For C#, the Azure Functions MCP extension supports the isolated worker model and ships as the `Microsoft.Azure.Functions.Worker.Extensions.Mcp` NuGet package. Clients connect to the Streamable HTTP endpoint at `/runtime/webhooks/mcp`. When hosted in Azure, the endpoint requires the system key named `mcp_extension` unless you configure the webhook authorization level as anonymous and rely on built-in MCP server authorization for identity-based access control.

```bash
az functionapp keys list --resource-group <RESOURCE_GROUP> --name <APP_NAME> --query systemKeys.mcp_extension --output tsv
```

The Azure Functions documentation [recommends managed identities where possible](https://learn.microsoft.com/en-us/azure/azure-functions/functions-identity-based-connections-tutorial) for remote-service connections, including host storage scenarios that support identity-based configuration. The MCP binding documentation also notes that the older Server-Sent Events transport is deprecated in newer protocol versions, so new server designs should prefer Streamable HTTP unless a specific client requires SSE compatibility.

## Why This Matters for Government

State and local agencies are modernizing long-lived line-of-business applications with lean teams, fixed budgets, and strict accountability requirements. AI coding agents can help, but only when their suggestions are grounded in approved sources and executed through auditable controls.

**Accuracy and auditability.** A grounding server that returns passages with source paths gives reviewers a trail back to approved engineering guidance, security runbooks, and architecture standards. That matters for code review, procurement evidence, and operational handoff.

**Controlled data boundaries.** A local stdio server keeps the MCP server process on the developer workstation. A remote server hosted in Azure Functions can run inside an Azure subscription with Microsoft Entra authentication patterns, managed identities, logging, and network controls. Teams still need to validate how their AI host handles returned context, but MCP lets the agency control the tools and data sources exposed to the agent.

**Compliance alignment.** Azure and Azure Government maintain [FedRAMP High authorizations](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-fedramp) for in-scope services, and Microsoft publishes guidance for [CJIS](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-cjis) and [IRS Publication 1075](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-irs-1075) workloads. Those documents do not automatically certify an agency application, but they help security teams map platform controls, customer-managed keys, identity, logging, and data residency decisions to agency obligations.

**Cloud-fit with parity checks.** [Azure Government uses the same underlying technologies as global Azure](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure), and Azure Functions is listed among Azure Government service endpoints. Microsoft also notes that some services and features can differ by cloud or region, so confirm MCP bindings, Functions hosting plans, and dependent services in the target cloud before committing to an architecture.

**Modernization velocity with guardrails.** A live-app MCP server shortens the fix-verify loop without turning the agent into an unrestricted operator. The best design is boring on purpose: small allow-listed tools, structured responses, explicit authorization, and logs that a team can review after the session.

## Getting started

Stand up the grounding server first. It delivers value quickly and carries the least risk because it is read-only. Point it at a small, high-value slice of approved documentation, wire it into GitHub Copilot agent mode with a local `mcp.json`, and measure whether the agent references current source paths more often and invents fewer APIs.

Then add the live-app server after the team is comfortable with the tool-approval flow and server-side authorization model. Both servers are ordinary C# projects, so they belong in source control, go through code review, and ship through the same pipelines as any other .NET service.

The pattern is durable: give agents grounded knowledge, expose live state through narrow tools, and keep humans and policy controls in the loop. That is how government .NET teams can use AI coding agents without asking reviewers to trust a guess.
