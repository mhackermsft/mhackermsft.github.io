---
title: 'SQL MCP Server: Building Secure Agentic Data Access for Government Workloads'
date: 2026-06-12T17:07:39+00:00
author: Mike Hacker
tags:
- AI
- Data Platform
- Security
categories:
- Azure
- Government
- Security
summary: How SQL MCP Server in Data API builder helps government teams give AI agents least-privilege, role-aware access to sensitive data without exposing the database directly.
draft: false
image_prompt: A secure government data platform architecture showing an AI agent connecting through SQL MCP Server and Data API builder to Azure SQL with Entra ID, managed identity, policy controls, and audit telemetry.
image: cover.png
audio: audio.mp3
---

Every IT leader in government feels the same two pulls at once. The first is the appeal of letting an AI agent reach into operational data: fewer screens, fewer custom reports, fewer one-off queries, and faster answers for staff and constituents. The second is the reasonable concern about what happens when a nondeterministic model starts writing SQL against a production system that holds citizen records. Those instincts are not in conflict. They are both correct, and the way to honor both is architecture.

That is the gap the **SQL MCP Server** in Data API builder (DAB) is designed to fill. SQL MCP Server is included in DAB starting with version 1.7, and Microsoft documentation describes DAB 2.0 as a public preview focused on MCP and AI integration. It gives agents a controlled, least-privilege contract to configured database entities without exposing the database directly or relying on natural-language-to-SQL generation. The Azure SQL team covered the design philosophy in May 2026 posts on Azure SQL Dev Corner: "Considering NL2SQL? Should your database really be the prompt? How can SQL MCP Server help?" and "Microsoft SQL Security Across the MAESTRO Stack: Building Secure Agentic AI with Defense-in-Depth."

## What the Model Context Protocol actually buys you

The [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) is an open standard for connecting AI applications to external systems. A tool is a described operation with defined inputs, outputs, and behavior. [SQL MCP Server](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/overview) implements MCP protocol version 2025-06-18 and supports two transports: streamable HTTP for hosted scenarios and stdio for local and CLI workflows.

The important architectural decision is what the server refuses to do. As the overview documentation states, SQL MCP Server intentionally does not support NL2SQL. Models are not deterministic, and the complex queries users most want AI to generate are the ones that can produce subtle, hard-to-catch errors. Instead, the server uses DAB's entity abstraction layer and built-in query builder. The agent works through structured tool calls, and DAB generates deterministic database queries for the configured source. For Microsoft SQL sources, that means well-formed Transact-SQL generated from structured inputs rather than raw SQL composed by the model.

## The tools an agent actually sees

Rather than handing an agent a SQL prompt, SQL MCP Server exposes a small, typed family of Data Manipulation Language (DML) tools, [documented in Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/data-manipulation-language-tools):

- `describe_entities` - discovers available entities, fields, and allowed operations from in-memory configuration
- `read_records` - filters, sorts, paginates, and projects from a single table or view
- `create_record` - inserts a row
- `update_record` - modifies a row by primary key
- `delete_record` - removes a row by primary key
- `execute_entity` - runs a stored procedure entity
- `aggregate_records` - performs count, sum, average, minimum, and maximum in DAB 2.0 preview

Notice what is missing. There is no DDL. SQL MCP Server works on the data plane, not on schema. Agents cannot create, alter, or drop objects through these DML tools. The `read_records` tool is intentionally scoped to a single table or view and does not support JOIN operations. For richer shapes, expose a database view or stored procedure, which keeps responsibility isolated and keeps the agent's context window smaller.

## Authentication: Entra ID at the front door, managed identity at the back

There are two trust boundaries to reason about.

**Agent to server.** Configure DAB with the Microsoft Entra ID JWT provider so inbound requests carry a validated bearer token. In `dab-config.json`:

```json
"runtime": {
  "host": {
    "mode": "production",
    "authentication": {
      "provider": "EntraID",
      "jwt": {
        "audience": "<expected-token-audience>",
        "issuer": "https://login.microsoftonline.com/<tenant-id>/v2.0"
      }
    }
  }
}
```

DAB assigns every request a role. Unauthenticated requests get the `Anonymous` system role. Valid tokens get `Authenticated`. To act as a specific user role, the client sends the `X-MS-API-ROLE` header with a role that is present in the token's `roles` claim. A request that names a role it does not hold is rejected with 403, as described in the [DAB authorization overview](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/authorization-overview).

**Server to database.** Keep passwords out of configuration by using managed identity for Azure SQL connections where appropriate. Microsoft SQL connection strings support the `Active Directory Managed Identity` authentication value:

```text
Server=tcp:<server>.database.windows.net,1433;
Database=<db>;
Authentication=Active Directory Managed Identity;
Encrypt=True;
```

DAB configuration values can also use [`@env()`](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/config/env-function) or [`@akv()`](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/config/akv-function) substitution so connection settings do not live directly in source control. For user-assigned managed identity scenarios, validate the required identity configuration for your hosting platform and database.

## Scoping read-only versus read-write

This is where guardrails get concrete. Permissions are defined per entity and per role, and DAB applies them consistently across REST, GraphQL, and MCP. A role only sees the entities, fields, rows, and operations it is granted. Here is a read-only public-records entity alongside a caseworker entity that may read, create, and update while excluding an SSN field:

```json
"entities": {
  "PermitApplication": {
    "source": "dbo.PermitApplications",
    "permissions": [
      { "role": "Anonymous", "actions": [ "read" ] }
    ]
  },
  "CaseRecord": {
    "source": "dbo.CaseRecords",
    "permissions": [
      {
        "role": "Caseworker",
        "actions": [ "read", "create", "update" ],
        "fields": { "exclude": [ "SSN" ] }
      }
    ]
  }
}
```

Because RBAC, field restrictions, and database policies are enforced for DAB operations, an agent operating as `Caseworker` cannot delete a case record or read the excluded `SSN` field through the configured entity. For row-level filtering, DAB database policies can add predicates for read, update, and delete actions, while SQL Server row-level security can be implemented through session context when that pattern is appropriate. The DML tools documentation also notes that you can disable a tool such as `delete_record` globally, while remembering that entity-level permissions remain the strongest control.

## A hands-on walkthrough

First, install the cross-platform CLI. The DAB CLI is distributed as a .NET tool and requires .NET 8:

```powershell
dotnet tool install --global Microsoft.DataApiBuilder
```

Then scaffold a configuration:

```powershell
dab init --database-type mssql --connection-string "@env('SQL_CONN')" --config dab-config.json --host-mode development
```

Add your entities and permissions, then start the runtime. MCP is enabled by default in current DAB configurations unless you explicitly disable it:

```powershell
dab start
```

The MCP endpoint comes up at `http://localhost:5000/mcp` by default. Before wiring an agent, inspect the live tool surface with the MCP Inspector, which proxies requests and helps avoid browser CORS and session-header issues:

```powershell
npx -y @modelcontextprotocol/inspector http://localhost:5000/mcp
```

Now wire an agent client. Semantic Kernel's current MCP plugin documentation shows the `semantic-kernel[mcp]` extra and MCP plugin classes, including streamable HTTP support:

```powershell
pip install "semantic-kernel[mcp]"
```

```python
from semantic_kernel import Kernel
from semantic_kernel.connectors.mcp import MCPStreamableHttpPlugin

async def main():
    async with MCPStreamableHttpPlugin(
        name="GovData",
        description="Read-only access to public permit records",
        url="http://localhost:5000/mcp",
    ) as data_plugin:
        kernel = Kernel()
        kernel.add_plugin(data_plugin)
        # Add your chat completion service and invoke the agent.
```

The agent can call `describe_entities` to learn the shape of `PermitApplication`, then call `read_records` with an OData-style filter. Tool calls are role-checked and traced. Results from `read_records` are automatically cached by DAB's caching system, while write operations remain permission-scoped operations against the configured entity.

For local-only experiments, you can skip HTTP and run stdio mode:

```powershell
dab start --mcp-stdio role:Caseworker
```

In stdio mode, DAB uses the simulator authentication provider, does not bind an HTTP port, and limits incoming requests to 1 MB. Use this for local development and direct agent integration, not as a production authentication pattern.

For production hosting, the May 2026 Azure SQL Dev Corner walkthrough "SQL MCP Server as an App Service" shows deployment without containers. DAB can also run on supported container platforms such as Azure Container Apps and Azure Kubernetes Service. Operations teams can use DAB support for OpenTelemetry, Application Insights, Azure Log Analytics, file logging, and health endpoints.

## Why This Matters for Government

Agencies hold some of the most sensitive data anywhere: benefits, case files, tax records, identities, health information, and public-safety data. The fear of pointing a model at that data is well founded, and SQL MCP Server is built around that fear rather than dismissing it.

**Least privilege is the default operating model.** An agent does not need direct database credentials, does not compose raw SQL, and does not see an entity or field outside its role. A public-facing constituent chatbot can be scoped to read-only `Anonymous` access on published records, while an internal caseworker agent gets a separate role with narrow write access and PII fields excluded. The configuration file becomes an auditable contract for what the agent can and cannot do.

**The pattern fits government cloud and compliance planning.** DAB is open source, free, and self-hosted. Microsoft documents DAB as running in containers across Azure, other clouds, and on-premises environments, and Azure Government provides dedicated US government regions with additional assurances for eligible government customers. For State and Local Government programs that evaluate StateRAMP, FedRAMP, CJIS, IRS Publication 1075, HIPAA, or agency-specific authorization requirements, the important point is architectural control: the data-access layer can be deployed inside the hosting environment and network boundary your agency authorizes. Confirm the availability and configuration differences for the agent host, App Service, container platform, database service, and monitoring services in the exact cloud and region you plan to use.

**Determinism is auditable.** The same structured tool call produces the same database query behavior for the configured entity and role. That predictability is what lets a CISO evaluate blast radius, an auditor reproduce a result, and a compliance officer reason about exactly what an agent can and cannot touch. Combined with Entra ID authentication, managed-identity database connections, Key Vault or environment-based configuration substitution, field restrictions, database policies, row-level security where appropriate, and full telemetry, agentic data access becomes a reviewable governed capability.

The takeaway for government technology leaders: you do not have to choose between the productivity of agents and the protection of citizen data. Put the guardrails in the data layer, verify every host and service against your compliance boundary, and let agents work inside a contract you can test and audit.

### Sources

- [SQL MCP Server overview - Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/overview)
- [DML tools reference - Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/data-manipulation-language-tools)
- [Data API builder authorization overview - Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/authorization-overview)
- [Data API builder CLI install - Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-api-builder/command-line/install)
- [Data API builder CLI init - Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-api-builder/command-line/dab-init)
- [Data API builder CLI start - Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-api-builder/command-line/dab-start)
- [Stdio transport for SQL MCP Server - Microsoft Learn](https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/stdio-transport)
- [Adding MCP plugins to Semantic Kernel - Microsoft Learn](https://learn.microsoft.com/en-us/semantic-kernel/concepts/plugins/adding-mcp-plugins)
- [Considering NL2SQL? Should your database really be the prompt? - Azure SQL Dev Corner](https://devblogs.microsoft.com/azure-sql/sql-mcp-server-nl2sql/)
- [Microsoft SQL Security Across the MAESTRO Stack - Azure SQL Dev Corner](https://devblogs.microsoft.com/azure-sql/microsoft-sql-security-across-the-maestro-stack-building-secure-agentic-ai-with-defense-in-depth/)
- [SQL MCP Server as an App Service - Azure SQL Dev Corner](https://devblogs.microsoft.com/azure-sql/sql-mcp-server-app-service/)
