---
title: 'Unity AI Gateway on Azure Databricks: A Single Control Plane for Governed AI in Government'
date: 2026-08-07T14:23:27+00:00
author: Mike Hacker
tags:
- Data Platform
- AI
- Security
categories:
- Azure Databricks
summary: A technical architecture guide to using Unity AI Gateway on Azure Databricks as one governed control plane for models, agents, tools, and MCP services, with routing, rate limiting, credential vaulting, observability, and near-real-time budget controls for government teams.
draft: false
image_prompt: Architecture diagram showing Azure Databricks Unity AI Gateway as a central control plane for governed public sector AI, connecting users, agents, models, MCP services, tools, Unity Catalog permissions, observability tables, and budget controls in a secure Azure environment.
image: cover.png
audio: audio.mp3
---

Public sector AI is no longer one chatbot pilot behind a single API key. It is coding agents in developer workflows, retrieval copilots over case records, and multi-step agents that call tools and external services. Each interaction is a place where the wrong model can be reached, a secret can leak, spend can run away, or an auditor can later ask a question the agency cannot answer.

Unity AI Gateway on Azure Databricks is designed to answer those questions from one place. The [Azure Databricks August 2026 release notes](https://learn.microsoft.com/en-us/azure/databricks/release-notes/product/2026/august) state that Unity AI Gateway became generally available on August 4, 2026, while some capabilities, including service policies and agent services, remain in Beta. The Microsoft Learn overview, [AI governance with Unity AI Gateway](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/), describes the gateway as the Azure Databricks governance solution for enterprise AI, built on Unity Catalog and extending governance beyond data and AI assets to runtime interactions between models, agents, MCP servers, and tools.

Availability is the first design gate. Unity AI Gateway requires a workspace in a supported region and is built on Unity Catalog. The Microsoft Learn page for [Azure Databricks features with limited regional availability](https://learn.microsoft.com/en-us/azure/databricks/resources/feature-region-support) currently states that Azure Government regions `usgovaz` and `usgovva` do not support Databricks SQL or Unity Catalog. Treat this architecture as an Azure commercial pattern unless your target cloud, region, compliance profile, and account configuration support Unity Catalog and Unity AI Gateway.

## The core idea: govern the runtime, not just the data

Most data platforms already govern tables and files. The AI-era gap is the runtime path a request travels: which model answered, which tool an agent called, whose credentials it used, and what it cost. Unity AI Gateway closes that gap by registering AI assets as Unity Catalog securable objects, then applying the same grant and revoke access model you already use for tables and volumes.

The Microsoft Learn overview describes four governed asset groups:

- **Models** served natively through Foundation Model APIs or external providers.
- **Agents** registered as Unity Catalog models.
- **MCP services** that expose tools to agents.
- **Custom tools and HTTP connections** that agents use to reach functions and external APIs.

Because requests route through one control plane, access, routing, guardrails, and observability can be managed together instead of being rebuilt in every application.

## Models: native serving plus governed external providers

Azure Databricks serves large language models natively through [Foundation Model APIs](https://learn.microsoft.com/en-us/azure/databricks/machine-learning/foundation-model-apis/), offering pay-per-token access with no infrastructure to run. For production, Microsoft documents two paths worth knowing: **priority pay-per-token**, which is opt-in per request by setting the `service_tier` parameter to `priority`, and **provisioned throughput**, which is recommended for production workloads and is available with compliance certifications such as HIPAA for workloads with performance or security requirements.

When you need a model that is not served natively, the current Unity AI Gateway pattern is to use a [model provider service](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/model-provider-services). A model provider service is a Unity Catalog securable object that stores encrypted provider credentials and lets callers query the provider without seeing the secret. The documented providers include OpenAI, Azure OpenAI, Anthropic, Amazon Bedrock, Microsoft Foundry, Google Gemini Enterprise, and custom bearer-token endpoints.

Microsoft also documents [external models in Model Serving](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/external-models/) for endpoints configured directly with providers. That page flags its MLflow Deployments CRUD API examples as Public Preview, so government teams should treat those examples as configuration guidance, not as a production automation standard. In either pattern, use Databricks secret references or model provider services rather than literal API keys in code.

## Routing and rate limiting: capacity and cost as policy

Once traffic is centralized, you can manage it. The Microsoft Learn overview lists rate limits, traffic splitting with fallbacks, and budgets. The [rate limits](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/rate-limits) page documents the mechanics:

- **Model services** support requests-per-minute (QPM) and tokens-per-minute (TPM) limits. **MCP services** support QPM only.
- Limits can be set at a global **Service** ceiling, a **User (Default)** level, and **Custom** limits for named users, service principals, or groups.
- User-specific limits take priority over group limits, custom limits override the default, and when both request and token limits apply, the more restrictive limit wins.
- When a caller exceeds a limit, the service returns **HTTP 429 (Too Many Requests)**, and clients should implement retry logic with exponential backoff.

Design around the documented limits: a maximum of 20 rate limits per service and 5 group-specific limits per service. The limiter records usage after a response is sent, so short bursts slightly above the configured limit can occur while the average request rate converges to the configured limit over time.

A practical government pattern is to set a conservative global Service ceiling on each production endpoint, a modest User (Default) so no single analyst drains capacity, and a higher Custom group limit for validated citizen-facing workloads. Add traffic splitting and fallbacks where supported so a provider outage can degrade to an approved alternate destination instead of failing outright.

## Guardrails: content policy at the request and response boundary

Routing controls who and how much. Service policies, which the overview also calls guardrails, control what content is allowed to proceed. Per Microsoft Learn, service policies control how each request and response proceeds based on its content and on who is making the call, using built-in and custom policies attached to a model or MCP service. The August 2026 release notes state that service policies remain in Beta, so agencies should evaluate preview terms, supportability, and authorization impact before making them mandatory in production.

A content policy attached to a service can apply consistently to every caller of that service, including agents calling on a user's behalf. For public sector records handling, that is the difference between a policy you can attest to and a policy you hope every app team remembered.

## MCP and agents: governing what tools an agent can reach

Agentic workloads are where governance often breaks, because an agent's blast radius is the set of tools it can call. On Azure Databricks, per the [MCPs and agent tools](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/agent-framework/mcp) page, MCP servers and tools are governed through Unity AI Gateway, with Unity Catalog enforcing permissions and managing credentials so agents and users reach only the tools and data you grant them.

Microsoft documents ready-to-use managed MCP servers for governed access to Genie, AI Search, Databricks SQL, and Unity Catalog functions, plus paths to connect external MCPs such as Slack, Google Drive, or any API through managed OAuth, the Unity Catalog connections proxy, or Unity Catalog function tools. The August 2026 release notes also state that Databricks-managed MCP connector integration with Unity AI Gateway is in Beta. The governance win is tool filtering: you grant an agent access to a specific set of MCP tools, and that grant is a Unity Catalog privilege you can audit and revoke.

## Observability and cost governance: the answers auditors ask for

Because every request routes through the gateway, observability is centralized by construction. The [Unity AI Gateway observability](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/observability) page documents the system tables that back it, and the broader [system tables reference](https://learn.microsoft.com/en-us/azure/databricks/admin/system-tables/) currently lists `system.ai_gateway.usage` and `system.ai_gateway.external_model_spend` as Beta and `system.access.audit` as Public Preview. Government teams should account for that preview status in production readiness and evidence planning.

Key tables and controls include:

- `system.ai_gateway.usage`, which logs token counts, latency, requester details, and request tags for requests routed through the gateway.
- `system.billing.usage` and `system.billing.list_prices`, which track consumption in Databricks units (DBUs) and list price so teams can estimate foundation model spend.
- `system.ai_gateway.external_model_spend`, which records estimated USD spend, aggregated hourly, for external-model requests through model provider services. Microsoft notes this figure is informational only.
- `system.access.audit`, which helps show who accessed which AI service and when. For agents, the `run_by` and `run_as` fields in `identity_metadata` distinguish a human caller from an agent acting on their behalf.

For cost governance, Microsoft's guidance is to attach **request or service tags** so you can attribute spend by business unit, project, or another dimension, then use the built-in usage dashboard's Cost Analysis view to track dollar spend by endpoint, model, or user. [Budgets for Unity AI Gateway](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/budgets) add shared thresholds, per-user thresholds, per-user overrides, alert actions, and block-usage actions.

Design those budget controls with the documented limitations in mind. Budgets provide a near-real-time estimate, not an absolute cap on final billed amounts. Unity AI Gateway budgets currently track pay-per-token and `ai_query` batch inference. Provisioned throughput, external-model inference, and model provider service usage are not currently tracked by budgets, so `system.billing.usage` remains the source of truth for actual billable usage.

Enable **inference tables** on a model service when you need to log request and response payloads to a Delta table for troubleshooting, guardrail validation, or sensitive-data exposure audits. One access-model detail to plan for: the observability docs state that access to these system tables requires both account and metastore admin roles by default, though admins can publish the usage dashboard to share analysis without granting raw table access.

## Why This Matters for Government

State and local agencies are being asked to adopt AI responsibly while proving control to auditors, oversight bodies, and residents. For programs that map to FedRAMP, CJIS, IRS Publication 1075, HIPAA, StateRAMP-aligned procurement, or local records-retention requirements, Unity AI Gateway does not replace the agency's authorization work. It can, however, make the evidence trail easier to assemble when the feature is available in the agency's target cloud and region.

- **One accountable control plane.** Access, routing, guardrails, spend, and audit attach to the same securables, so you can revoke a model or tool through Unity Catalog governance.
- **Auditable identity, including agents.** The `run_by` and `run_as` distinction helps show whether a human or an agent acting for them made a call.
- **Cost discipline built in.** Tags, rate limits, budget thresholds, alerts, and block-usage actions help forecast and contain AI spend. Pair those controls with billing-table reconciliation because budget enforcement is approximate.
- **Credential hygiene by default.** Secret-referenced keys and model provider services keep provider credentials out of code and out of end-user hands, reducing a common breach path.
- **Compliance-aware deployment choices.** The Foundation Model APIs documentation notes HIPAA-related compliance certification availability for provisioned throughput, while Microsoft Azure Government documentation explains that cloud and feature availability can differ from global Azure.

A closing caution worth repeating for this audience: confirm availability before you commit. Azure Databricks features can vary by region, Unity AI Gateway requires supported regions and Unity Catalog, and Azure Government has distinct service and endpoint boundaries. Start from the [Unity AI Gateway overview](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/), the Azure Databricks regional availability documentation, and your Microsoft or Databricks account team before you design the control plane your agency will depend on.

## Get started

- [AI governance with Unity AI Gateway](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/) - the end-to-end governance guide.
- [Apply rate limits to model and MCP services](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/rate-limits) - QPM/TPM controls and precedence.
- [Unity AI Gateway observability](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/observability) - system tables, cost analysis, and audit.
- [Manage budgets for Unity AI Gateway](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/budgets) - thresholds, alerts, block-usage actions, and budget limitations.
- [Govern external model providers](https://learn.microsoft.com/en-us/azure/databricks/ai-gateway/model-provider-services) - BYOK provider services as Unity Catalog securables.
- [External models in Model Serving](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/external-models/) - endpoint configuration and provider credential references.
- [MCPs and agent tools](https://learn.microsoft.com/en-us/azure/databricks/generative-ai/agent-framework/mcp) - governing the tools your agents can reach.
- [Azure Databricks features with limited regional availability](https://learn.microsoft.com/en-us/azure/databricks/resources/feature-region-support) - region and cloud planning constraints.
