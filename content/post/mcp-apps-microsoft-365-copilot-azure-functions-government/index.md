---
title: 'MCP Apps in Microsoft 365 Copilot: Building Custom AI-Powered Experiences with Azure Functions'
date: 2026-04-10T13:43:08+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- How To
categories:
- AI
summary: Microsoft 365 Copilot now supports MCP Apps, enabling government organizations to build custom, interactive AI-powered app experiences directly in Copilot chat using Azure Functions and the Model Context Protocol.
draft: false
image_prompt: A large brass skeleton key hovering above a complex arrangement of interlocking bronze gears and clockwork mechanisms, with warm golden light streaming through the gaps between the gears casting dramatic shadows. A glowing crystal prism sits at the center of the gear assembly, refracting the light into brilliant spectral rays. The scene is photographed from a low angle with shallow depth of field, giving the mechanical assembly a monumental, architectural quality. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
audio: audio.mp3
---

The way government employees interact with AI assistants is about to change significantly. On April 7, 2026, Microsoft [announced that MCP Apps are now available in Microsoft 365 Copilot chat](https://devblogs.microsoft.com/microsoft365dev/mcp-apps-now-available-in-copilot-chat/), bringing rich, interactive app experiences directly into conversational AI workflows. For state and local government organizations already investing in Microsoft 365 Copilot, this opens up powerful new possibilities for building custom line-of-business tools that meet employees exactly where they work.

Built on the open-source [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) standard and readily deployable through [Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-mcp), MCP Apps represent a fundamental shift from simple text-based AI responses to fully interactive application experiences embedded within Copilot.

## What Are MCP Apps?

The Model Context Protocol is an open standard for connecting AI applications to external systems, tools, and data sources. Think of it as a universal adapter for AI: just as USB-C provides a standardized way to connect devices, MCP provides a standardized way for AI models to discover and use external tools.

MCP Apps take this a step further. They extend the MCP standard to allow servers to deliver **interactive user interfaces** directly within the AI host. In the context of Microsoft 365 Copilot, this means agents can now present tables, forms, maps, rich media, and specialized interfaces, all rendered securely in a sandboxed iframe within the chat experience.

As [Microsoft described in the announcement](https://techcommunity.microsoft.com/blog/microsoft365copilotblog/enable-agents-to-bring-apps-into-the-flow-of-work-while-keeping-it-in-control/4499464), agents built with MCP Apps "go well beyond text responses" and can "present tables, forms, diagrams, dashboards, maps, rich media, and specialized creation surfaces, all securely rendered in a sandboxed iFrame within chat."

There are two complementary display modes:

- **Inline mode** displays lightweight widgets directly in the conversation, ideal for quick interactions like previews, simple actions, or confirmations
- **Side-by-side mode** provides an expanded workspace alongside the conversation for complex workflows like multistep editing or extended review tasks

## Why Azure Functions for MCP Servers?

Azure Functions provides a natural hosting platform for MCP servers, and Microsoft has released a dedicated [Azure Functions MCP extension](https://learn.microsoft.com/en-us/azure/azure-functions/functions-bindings-mcp) that makes building remote MCP servers straightforward.

Here is why Azure Functions is a compelling choice for government organizations:

**Serverless and cost-efficient.** MCP servers built on the Azure Functions Flex Consumption plan follow a pay-for-what-you-use billing model. Government agencies only pay when their MCP tools are actually invoked, which is ideal for tools that may see intermittent usage patterns.

**Multi-language support.** The MCP extension supports C#, Java, Python, TypeScript, and JavaScript, letting development teams use the languages they already know. The extension provides trigger bindings for both MCP tools (the logic) and MCP resources (the UI).

**Built-in security.** Azure Functions integrates with Microsoft Entra ID for authentication, supports OAuth 2.1 and single sign-on (SSO), and provides system-level keys for endpoint protection. The extension requires a system key named `mcp_extension` for hosted endpoints, with options to configure authorization levels through the `host.json` settings.

**Two transport options.** The extension supports both Streamable HTTP (recommended) and Server-Sent Events (SSE) transports, giving teams flexibility in how clients connect to their MCP servers.

## How It Works: The Architecture

An MCP App built with Azure Functions consists of two key components:

1. **A tool trigger function with UI metadata** that handles the business logic and declares a `ui.resourceUri` pointing to a UI resource
2. **A resource trigger function** that serves the bundled HTML and JavaScript at the matching URI

Here is a simplified example of what this looks like in C#:

```csharp
// Tool function with UI metadata
[Function(nameof(GetPermitStatus))]
public async Task<object> GetPermitStatus(
    [McpToolTrigger(nameof(GetPermitStatus), 
        "Returns permit status for a given application number.")]
    [McpMetadata(ToolMetadata)]
    ToolInvocationContext context,
    [McpToolProperty("permitNumber", "Permit application number")]
    string permitNumber)
{
    var result = await _permitService.GetStatusAsync(permitNumber);
    return result;
}

// Resource function serving the interactive UI
[Function(nameof(GetPermitWidget))]
public string GetPermitWidget(
    [McpResourceTrigger(
        "ui://permits/index.html",
        "Permit Status Widget",
        MimeType = "text/html;profile=mcp-app",
        Description = "Interactive permit status display")]
    [McpMetadata(ResourceMetadata)]
    ResourceInvocationContext context)
{
    var file = Path.Combine(AppContext.BaseDirectory, 
        "app", "dist", "index.html");
    return File.ReadAllText(file);
}
```

The tool metadata declares the UI resource URI, and the resource trigger serves the interactive frontend. When a user asks Copilot about a permit status, the agent invokes the tool, gets the data, and renders the interactive widget directly in the chat.

## Getting Started with Development

Microsoft provides multiple paths to get started:

### Using the Microsoft 365 Agents Toolkit

The [Microsoft 365 Agents Toolkit](https://marketplace.visualstudio.com/items?itemName=TeamsDevApp.ms-teams-vscode-extension) for Visual Studio Code (version 6.6.1 or later) provides a streamlined experience. Developers can create a new declarative agent, select "Add an Action," choose "Start with an MCP Server," and provide the URL for their MCP server.

The toolkit handles generating the required manifest files and provides a sideloading capability for testing agents directly in [Microsoft 365 Copilot chat](https://m365.cloud.microsoft/chat).

### Using Azure Developer CLI Templates

For building the Azure Functions MCP server itself, Microsoft provides [quickstart templates](https://learn.microsoft.com/en-us/azure/azure-functions/scenario-mcp-apps) through the Azure Developer CLI (`azd`). These templates scaffold a complete MCP Apps project with a sample weather widget, and are available for C#, TypeScript, Python, and Java.

```bash
azd init --template remote-mcp-functions-dotnet -e my-mcp-app
```

### Sample Projects

Microsoft has published several [interactive UI samples on GitHub](https://github.com/microsoft/mcp-interactiveUI-samples), including:

- **Field Service Dispatch** with map visualization and dispatch planning
- **HR Consultant Management** with Fluent UI React widgets including dashboards and profile cards
- **Employee Training** with embedded video previews and course views

These samples demonstrate real-world patterns that government developers can adapt.

## Governance and Security

A critical consideration for government IT leaders: MCP Apps agents follow the **same governance, security, and administrative controls** as other declarative agents in your Microsoft 365 environment. This means:

- IT administrators control which agents are available in the tenant and who can use them
- Each agent operates within existing app permissions and identity boundaries
- Agents can be monitored end-to-end using [Agent 365](https://learn.microsoft.com/en-us/microsoft-agent-365/), Microsoft's unified control plane for agent management
- [Agent Evaluations](https://learn.microsoft.com/en-us/microsoft-copilot-studio/analytics-agent-evaluation-intro) in Microsoft Copilot Studio provide structured quality assessment, running agents against test scenarios and generating objective scores for accuracy and intent alignment

Agents can be deployed through the Microsoft 365 Admin Center, scoped to specific users or groups, and published internally or to the broader Microsoft 365 Agent Store.

## Why This Matters for Government

State and local government organizations stand to benefit significantly from MCP Apps for several reasons:

**Streamlined citizen-facing workflows.** Government employees often work with complex, multi-step processes like permit applications, case management, benefits enrollment, and inspection scheduling. MCP Apps can bring interactive forms and status dashboards directly into the Copilot chat, eliminating the need to switch between multiple legacy applications.

**Reduced context switching.** A caseworker who needs to look up a resident's case history, update a record, and schedule a follow-up can do all of this within a single Copilot conversation. The interactive UI surfaces the right data and actions without leaving the chat window.

**Incremental modernization of legacy systems.** Many government agencies maintain older line-of-business applications that are difficult and costly to replace. MCP servers can act as a modern interface layer on top of existing APIs and databases, giving employees an AI-powered front door to legacy systems without requiring a full rewrite.

**Built on open standards.** MCP is an open protocol, not a proprietary lock-in. Government organizations can build MCP servers that work with Microsoft 365 Copilot today and potentially integrate with other MCP-compatible AI clients in the future.

**Azure Government compatibility.** Azure Functions is available in [Azure Government](https://learn.microsoft.com/en-us/azure/azure-government/documentation-government-welcome) regions, enabling agencies to host their MCP servers within the compliance boundaries required for government workloads. This is particularly relevant for organizations that require data residency within government cloud infrastructure.

**A note on availability:** As with many new Microsoft 365 features, MCP Apps and the associated Copilot extensibility capabilities are initially rolling out to commercial tenants. Government organizations operating in GCC, GCC High, or DoD environments should check the [Microsoft 365 Government roadmap](https://learn.microsoft.com/en-us/office365/servicedescriptions/office-365-platform-service-description/office-365-us-government/office-365-us-government) for the latest availability timeline. However, the Azure Functions MCP extension for building the server-side components can be developed and tested today in both Azure commercial and Azure Government environments, ensuring your team is ready when full support arrives.

## Practical Use Cases for Government

Consider these scenarios where MCP Apps could transform government workflows:

- **311 Service Requests:** An agent that lets employees look up, create, and update service requests with an interactive map widget showing locations and status
- **Budget and Procurement:** Interactive approval workflows rendered directly in Copilot, with forms pre-populated from Work IQ context like email threads and meeting notes
- **Inspections and Compliance:** Field inspection forms with photo upload capabilities and real-time status tracking, accessible through natural language prompts
- **HR and Onboarding:** Interactive employee onboarding checklists and training resource browsers embedded in Copilot conversations

## Next Steps

If your organization is exploring MCP Apps, here are the recommended first steps:

1. **Explore the documentation:** Start with the [MCP Apps interactive UI guide](https://learn.microsoft.com/en-us/microsoft-365-copilot/extensibility/declarative-agent-ui-widgets) on Microsoft Learn
2. **Try the quickstart:** Deploy the [Azure Functions MCP Apps template](https://learn.microsoft.com/en-us/azure/azure-functions/scenario-mcp-apps) to see the end-to-end developer experience
3. **Review the samples:** Examine the [interactive UI samples on GitHub](https://github.com/microsoft/mcp-interactiveUI-samples) for patterns you can adapt
4. **Identify a pilot workflow:** Choose a high-value, well-understood internal process as your first MCP Apps project
5. **Engage your Microsoft account team:** Discuss GCC availability timelines and get guidance specific to your environment

MCP Apps represent a meaningful step forward in how AI assistants can serve government employees. By combining the open Model Context Protocol standard with the serverless scalability of Azure Functions and the enterprise governance of Microsoft 365 Copilot, government organizations now have a clear path to building AI-powered interactive experiences that streamline the workflows their employees depend on every day.
