---
title: 'Azure MCP Server 2.0 GA: Building Self-Hosted Agentic Cloud Automation Workflows on Azure for Government'
date: 2026-04-13T14:41:05+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- Announcements
- How To
categories:
- AI
summary: Azure MCP Server 2.0 reaches general availability with 276 tools across 50+ Azure services, remote HTTP deployment on Azure Container Apps, sovereign cloud support for Azure Government, and comprehensive security hardening for agentic cloud automation workflows.
draft: false
image_prompt: A massive bronze compass resting on a polished marble surface, its needle pointing decisively forward. Around the compass, interlocking steel gears of various sizes turn in synchronized motion, connected by taut copper cables. A single golden key lies beside the compass, catching dramatic side lighting that casts long shadows across the marble. The scene is bathed in cool blue and warm amber tones from opposing directional lights, creating a cinematic chiaroscuro effect. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
audio: audio.mp3
---

On April 10, 2026, Microsoft released **Azure MCP Server 2.0.0** to general availability, a major milestone for teams building AI-powered cloud automation. This second major release delivers remote HTTP deployment, sovereign cloud support (including Azure US Government), 276 tools spanning 50+ Azure services, and deep security hardening designed for enterprise and government environments.

For government IT leaders and developers already exploring agentic AI, this release represents a concrete, production-ready path to building automation workflows where AI agents can safely interact with Azure resources on your behalf, all while respecting RBAC boundaries and sovereign cloud requirements.

## What Is the Azure MCP Server?

The [Azure MCP Server](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/overview) implements the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction), an open standard for connecting AI applications to external systems. Think of MCP as a universal adapter between AI agents and the tools they need to do real work. The Azure MCP Server provides that adapter specifically for Azure, exposing operations across compute, storage, databases, security, networking, AI services, and more as structured tools that AI agents can discover, understand, and invoke.

The server works with a broad set of MCP-compatible clients: [GitHub Copilot agent mode in VS Code](https://code.visualstudio.com/docs/copilot/chat/mcp-servers), the [OpenAI Agents SDK](https://openai.github.io/openai-agents-python/mcp/), [Semantic Kernel](https://learn.microsoft.com/en-us/semantic-kernel/), Cursor, Windsurf, Claude Desktop, and custom applications built with the [GitHub Copilot SDK](https://github.com/github/copilot-sdk).

## What Changed in 2.0: The Key Capabilities

### Remote HTTP Deployment

The biggest architectural change in 2.0 is the ability to deploy the Azure MCP Server as a **shared, multi-user HTTP service** with [Entra ID authentication and On-Behalf-Of (OBO) authorization](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/how-to/deploy-remote-mcp-server-microsoft-foundry). In version 1.x, the server ran locally via `stdio`, which was fine for individual developer workstations but didn't work for centralized agent platforms like [Microsoft Foundry](https://azure.microsoft.com/products/ai-foundry) or [Copilot Studio](https://www.microsoft.com/microsoft-copilot/microsoft-copilot-studio).

Now, you can deploy the server to [Azure Container Apps](https://learn.microsoft.com/en-us/azure/container-apps/overview) using an `azd` template, and your Foundry agents or Copilot Studio agents can securely call MCP tools over HTTPS. This opens the door to building centralized, governed automation services where multiple agents and users share a managed MCP endpoint.

Here's how to deploy in under five minutes:

```bash
# Clone the azd template for Microsoft Foundry integration
azd init -t azmcp-foundry-aca-mi

# Provision and deploy
azd up
```

The template provisions:
- **Azure Container App** running the MCP server with your chosen tool namespaces
- **Managed identity** with appropriate RBAC roles on target resources
- **Entra ID app registration** with OAuth 2.0 for client authentication
- **Application Insights** for telemetry and monitoring

After deployment, retrieve your endpoint:

```bash
azd env get-values
# Output includes:
# CONTAINER_APP_URL="https://azure-mcp-storage-server.<name>.eastus2.azurecontainerapps.io"
# ENTRA_APP_CLIENT_ID="<your-app-client-id>"
```

### Sovereign Cloud Support: Azure US Government

For government customers, **sovereign cloud support** is a critical requirement. Azure MCP Server 2.0 supports [Azure US Government](https://learn.microsoft.com/en-us/azure/azure-government/) natively. You can connect by setting the `--cloud` flag or environment variable:

```powershell
# Set the cloud target via environment variable
$env:AZURE_CLOUD = "AzureUSGovernment"
azmcp server start

# Or pass it as a command-line flag
azmcp server start --cloud AzureUSGovernment
```

Before connecting, authenticate your local tools against the Government cloud:

```powershell
# Azure CLI
az cloud set --name AzureUSGovernment
az login

# Azure PowerShell
Connect-AzAccount -Environment AzureUSGovernment
```

The supported cloud aliases are case-insensitive and include `AzureUSGovernment`, `USGov`, `AzureUSGovernmentCloud`, and `USGovernment`. For full configuration, see the [Sovereign Clouds documentation](https://github.com/microsoft/mcp/blob/main/docs/sovereign-clouds.md).

### 276 Tools Across 50+ Services

Version 2.0 expanded tool coverage from 170+ to **276 individual tools** across more than 50 Azure services. New namespaces added in 2.0 include:

| Namespace | Description |
|---|---|
| `advisor` | Azure Advisor recommendations |
| `compute` | VMs, VMSS, managed disks (create, update, delete) |
| `containerapps` | Azure Container Apps management |
| `functions` | Azure Functions listing and management |
| `policy` | Azure Policy operations |
| `pricing` | Azure pricing lookups |
| `azuremigrate` | Platform Landing Zone generation |
| `wellarchitectedframework` | Well-Architected reviews |
| `servicefabric` | Service Fabric cluster management |
| `storagesync` | Azure Storage Sync |
| `fileshares` | Azure managed file shares |
| `deviceregistry` | Azure Device Registry |

You can scope a server instance to specific namespaces using the `--namespace` flag, which is important for the principle of least privilege:

```bash
# Start server with only storage and keyvault tools
azmcp server start --namespace storage --namespace keyvault

# Start in read-only mode for monitoring agents
azmcp server start --read-only
```

### Security Hardening

The 2.0 release includes substantial security work that matters for government deployments:

- **Input validation and SSRF protection** across all service tools
- **SQL and KQL injection prevention** with query parameterization for MySQL, PostgreSQL, Cosmos DB, and Kusto-based tools
- **Endpoint validation** for Azure Blob Storage, Service Bus, and compute endpoints
- **User confirmation prompts (elicitation)** for sensitive and destructive operations, including Key Vault secret access and resource deletion
- **Tool annotations** that mark each tool as read-only, destructive, idempotent, or secret-handling, giving MCP clients the metadata to enforce safety policies
- **Read-only mode** (`--read-only`) that prevents all write operations, useful for monitoring-only agent deployments

### Performance Improvements

Server startup dropped from approximately 20 seconds to **1-2 seconds** when proxied MCP servers are enabled. The Docker images also got roughly 60% smaller through trimmed binaries, and both AMD64 and ARM64 images are now available.

## Architecture Pattern: Self-Hosted Agentic Automation for Government

Here is a practical architecture for government teams looking to build centralized, governed AI automation using Azure MCP Server 2.0:

**Layer 1: MCP Server on Azure Container Apps**
Deploy the Azure MCP Server as a remote HTTP endpoint using the [azd template](https://github.com/Azure-Samples/azmcp-foundry-aca-mi). Scope it to the specific namespaces your agents need (e.g., `storage`, `compute`, `monitor`). Enable `--read-only` for monitoring agents; leave write access for provisioning agents.

**Layer 2: Entra ID and RBAC**
The container app uses a managed identity with specific RBAC role assignments. The Entra app registration enforces OAuth 2.0 with the `Mcp.Tools.ReadWrite.All` role. This means your agents operate with the same identity and authorization model as any other Azure service principal. No shared keys, no over-permissioned service accounts.

**Layer 3: Agent Platform**
Connect from [Microsoft Foundry](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/how-to/deploy-remote-mcp-server-microsoft-foundry), [Copilot Studio](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/how-to/deploy-remote-mcp-server-copilot-studio), or custom applications built with the [GitHub Copilot SDK](https://github.com/github/copilot-sdk) or [Semantic Kernel](https://learn.microsoft.com/en-us/semantic-kernel/). The agent authenticates to the MCP server using its managed identity, then calls Azure tools on behalf of the user.

**Layer 4: Monitoring and Compliance**
Application Insights captures telemetry from the MCP server. Tool annotations and elicitation prompts provide an audit trail of which tools were invoked, whether they were destructive, and whether user confirmation was obtained.

## Code Example: Building a Custom Automation Agent in Python

Here is a concrete example using the [Azure MCP Server with Python](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/get-started/languages/python) to build an agent that can query Azure resources:

```python
import asyncio
from copilot import CopilotClient
from copilot.generated.session_events import SessionEventType

async def main():
    client = CopilotClient({
        "cli_args": ["--allow-all-tools", "--allow-all-paths"]
    })
    await client.start()

    azure_mcp_config = {
        "azure-mcp": {
            "type": "local",
            "command": "npx",
            "args": ["-y", "@azure/mcp@2.0.0", "server", "start",
                     "--cloud", "AzureUSGovernment"],
            "tools": ["*"],
        }
    }

    session = await client.create_session({
        "model": "gpt-4.1",
        "streaming": True,
        "mcp_servers": azure_mcp_config,
    })

    def handle_event(event):
        if event.type == SessionEventType.ASSISTANT_MESSAGE_DELTA:
            if hasattr(event.data, 'delta_content') and event.data.delta_content:
                print(event.data.delta_content, end="", flush=True)
        elif event.type == SessionEventType.TOOL_EXECUTION_START:
            tool_name = getattr(event.data, 'tool_name', 'unknown')
            print(f"\n[Calling Azure tool: {tool_name}]")

    session.on(handle_event)

    await session.send_and_wait({
        "prompt": "List all resource groups and check for any "
                  "Azure Advisor recommendations related to security"
    })

    await client.stop()

if __name__ == "__main__":
    asyncio.run(main())
```

Note the `--cloud AzureUSGovernment` flag in the MCP server args, which directs all tool calls to the Government cloud endpoints.

## Installation Options

Azure MCP Server 2.0 is available through multiple package managers:

```bash
# .NET (NuGet)
dotnet tool install Azure.Mcp --version 2.0.0

# Node.js (NPM)
npx @azure/mcp@2.0.0 server start

# Python (PyPI)
uvx --from msmcp-azure azmcp server start

# Docker
docker pull mcr.microsoft.com/azure-sdk/azure-mcp:2.0.0
```

For IDE users, extensions are available for [VS Code](https://marketplace.visualstudio.com/items?itemName=ms-azuretools.vscode-azure-mcp-server), Visual Studio 2022/2026, IntelliJ IDEA, and Eclipse.

## Why This Matters for Government

**Sovereign cloud support is not optional for government.** Azure MCP Server 2.0's native Azure US Government support means agencies can build agentic automation workflows that stay within their compliance boundary. Unlike custom integrations that require manual endpoint configuration, the `--cloud AzureUSGovernment` flag handles endpoint routing automatically across all 276 tools.

**RBAC-based authorization aligns with Zero Trust.** Every tool call flows through Entra ID and Azure RBAC. There are no embedded credentials, no shared API keys. Managed identities authenticate the server to Azure resources, and OBO authorization ensures agents act with the delegated permissions of the requesting user.

**Audit and control are built in.** Tool annotations mark every operation as read-only, destructive, or secret-handling. Elicitation prompts require user confirmation before sensitive operations execute. The `--read-only` flag lets you deploy monitoring-only agents that cannot modify resources. Application Insights provides centralized telemetry. This is the kind of observability and control that government compliance frameworks demand.

**Operational efficiency at scale.** Government IT teams are often understaffed relative to their infrastructure footprint. An AI agent that can query Azure Monitor logs, check Advisor recommendations, list resource configurations, and generate CLI commands from natural language saves hours of manual portal navigation. When deployed as a shared remote server, multiple teams and agents can access a single governed endpoint.

**No vendor lock-in on the protocol.** MCP is an [open standard](https://modelcontextprotocol.io/introduction) supported by multiple AI platforms. Your investment in MCP-based automation workflows is portable across different AI clients and models.

## Getting Started

1. **Try it locally**: Install via your preferred package manager and connect from VS Code with GitHub Copilot agent mode
2. **Target Government cloud**: Set `AZURE_CLOUD=AzureUSGovernment` and authenticate with `az cloud set --name AzureUSGovernment && az login`
3. **Deploy remotely**: Use the [azd template](https://github.com/Azure-Samples/azmcp-foundry-aca-mi) to deploy to Azure Container Apps
4. **Scope and harden**: Use `--namespace` to limit tools, `--read-only` for monitoring agents, and configure RBAC on the managed identity

For the complete release notes, see the [Azure MCP Server 2.0.0 release on GitHub](https://github.com/microsoft/mcp/releases). For documentation, visit the [Azure MCP Server docs on Microsoft Learn](https://learn.microsoft.com/en-us/azure/developer/azure-mcp-server/).
