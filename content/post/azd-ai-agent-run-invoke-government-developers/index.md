---
title: 'Testing and Invoking Hosted AI Agents from the Terminal: A Practical Guide for Government Developers Using Azure Developer CLI'
date: 2026-03-16T17:15:18+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- How To
- Announcements
categories:
- Azure AI Foundry
- Azure Developer CLI
summary: New azd ai agent run and azd ai agent invoke commands in the Azure Developer CLI give government developers fast, terminal-based tools for local testing and deployment validation of hosted AI agents, shortening the debugging loop and improving reliability of constituent-facing AI services.
draft: false
image_prompt: A government developer at a terminal running Azure Developer CLI commands to test and validate an AI agent, with a clean modern workspace and Azure AI Foundry branding visible on a secondary monitor.
image: cover.png
---

Government IT teams are being asked to do something genuinely new: build and operate AI agents that serve real constituents. Whether it is a permit inquiry chatbot, a benefits eligibility assistant, or an internal document summarizer for agency staff, these AI-powered services carry expectations of reliability that traditional web apps once monopolized. When they fail, people notice.

For developers building and maintaining hosted AI agents on Microsoft Azure AI Foundry, the ability to test an agent locally before deploying - and to invoke a deployed agent directly from the terminal to validate behavior - has historically required context-switching between SDKs, REST clients, and portal playgrounds. In March 2026, the Azure Developer CLI (`azd`) team changed that with two new commands designed specifically for the hosted agent development loop: `azd ai agent run` and `azd ai agent invoke`. This post is a practical walkthrough of those commands and what they mean for government development teams.

---

## What Are Hosted AI Agents?

[Azure AI Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview) is a fully managed platform for building, deploying, and scaling AI agents. It supports three agent types:

- **Prompt agents** - Defined entirely through configuration, no code required.
- **Workflow agents (preview)** - Orchestrate multi-step automation using declarative definitions, buildable visually in the Foundry portal or via YAML in Visual Studio Code.
- **Hosted agents (preview)** - Code-based agents built with frameworks like Agent Framework, LangGraph, or custom code, deployed as containers on managed infrastructure.

Hosted agents are where government developers gain the most power - and the most complexity. You write the orchestration logic, package it as a container, and deploy it to Foundry, which handles scaling, identity, conversation management, and observability. Standard container development challenges apply: catching logic errors before deployment, validating tool-calling behavior, confirming a redeploy worked correctly. The faster you can test and validate, the shorter your iteration loop.

> **Note on Preview Availability:** Hosted agents are currently in public preview. Azure Government regions have different feature availability timelines than Azure commercial. Before planning a production deployment of hosted agents in Azure Government, verify current regional availability in the [Azure AI Foundry limits and quotas documentation](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts/limits-quotas-regions) and confirm support with your Microsoft account team.

---

## The `azd` AI Agent Extension

The commands in this post are part of the [`azure.ai.agents` extension](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/extensions/azure-ai-foundry-extension) for the Azure Developer CLI (`azd`). This extension brings agent-centric workflows directly into the terminal, complementing the standard `azd` lifecycle commands (`azd init`, `azd up`, `azd deploy`).

The extension handles the full agent development lifecycle:

- **`azd ai agent init`** - Scaffold an agent project from a manifest or GitHub template.
- **`azd ai agent run`** - Run your agent locally for development and testing before deploying.
- **`azd ai agent invoke`** - Invoke a deployed agent via the Responses API to validate behavior post-deployment.

The `run` and `invoke` commands were introduced in extension version `0.1.14-preview`, released March 10, 2026. Release details are available in the [Azure Developer CLI GitHub repository releases](https://github.com/Azure/azure-dev/releases).

### Prerequisites

To follow along, you need:

1. `azd` installed ([Install guide](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd)) - version 1.23.8 or later.
2. The AI agents extension installed or upgraded:

```bash
# Install the extension
azd extension install azure.ai.agents

# Or upgrade if already installed
azd extension upgrade azure.ai.agents
```

3. An authenticated `azd` session: `azd auth login`
4. A hosted agent project initialized in your local environment. If you are starting fresh, see the [hosted agent quickstart](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent).

---

## `azd ai agent run`: Test Your Agent Locally Before Deploying

One of the highest-value practices in any software deployment is catching problems before they reach production. For government teams operating under change management processes, a failed deployment is not just a technical inconvenience - it is a process event that may require documentation, approval, and follow-up review. Catching a bug locally before opening a change request is always preferable.

The `azd ai agent run` command starts your agent in your local environment, using the same configuration it will use when deployed to Foundry. You can interact with it, test tool calls, and validate behavior against your local development data - all without deploying a container.

### Usage

```bash
# Run the agent locally
azd ai agent run --name my-agent
```

The command brings up your agent using the project configuration defined in `azure.yaml`, wiring in the environment variables and connection settings that Foundry will use in production. You can send test queries, observe the agent's reasoning and tool calls, and confirm the behavior before committing to a deployment.

For government developers, this matters at every stage of the development cycle: initial development, post-change validation, and pre-release testing under change management constraints. Running locally is faster than deploying, and it does not consume cloud resources or trigger change-tracked deployment events.

---

## `azd ai agent invoke`: Validate a Deployed Agent from the Terminal

Once an agent is deployed, confirming it is working correctly should be as frictionless as possible. The `azd ai agent invoke` command calls your deployed agent via the Responses API directly from the terminal, without requiring a browser session in the Foundry portal or a separate REST client.

### Usage

```bash
# Invoke the deployed agent with a test query
azd ai agent invoke --name permit-inquiry-agent --input "What documents are required for a building permit?"
```

The command sends the input to the deployed agent, returns the response, and optionally surfaces trace information depending on the agent's observability configuration. This makes it a natural post-deployment validation step: deploy with `azd deploy`, then immediately invoke with `azd ai agent invoke` to confirm the agent is responding correctly.

### When to use `run` vs. `invoke`

| Command | What it does | When to use it |
|---|---|---|
| `azd ai agent run` | Runs the agent locally | Development, pre-deployment testing |
| `azd ai agent invoke` | Calls the deployed agent via Responses API | Post-deployment validation, smoke tests |

---

## A Practical Workflow for Government Development Teams

Your agency is developing a constituent-facing permit inquiry agent. Here is how `run` and `invoke` fit into the development and deployment workflow:

**Step 1 - Develop and test locally:**

```bash
# Initialize the project if you haven't already
azd ai agent init -m https://github.com/microsoft-foundry/foundry-samples/blob/main/samples/python/hosted-agents/agent-framework/agent-with-foundry-tools/agent.yaml

# Run locally to test behavior before deploying
azd ai agent run --name permit-inquiry-agent
```

Send test queries, observe tool calls, and confirm the agent responds correctly. Catch logic errors, misconfigured environment variables, and model behavior issues before they reach a deployment.

**Step 2 - Deploy to Foundry:**

```bash
azd deploy
```

**Step 3 - Validate the deployment:**

```bash
# Immediately after deploying, confirm the agent is responding correctly
azd ai agent invoke --name permit-inquiry-agent --input "What permits are required for a home addition?"
```

If the agent responds correctly, the deployment is validated. If it does not, you have a fast, terminal-based confirmation of a problem without navigating to the Foundry portal - and you can iterate immediately using `azd ai agent run` to diagnose the issue locally.

Total time from deploy to validated: under two minutes, from the same terminal session where you manage your entire deployment pipeline.

---

## Integrating into CI/CD Pipelines

Government development teams operating under NIST SP 800-53 or StateRAMP controls often need automated post-deployment validation. The `azd ai agent invoke` command is well suited to pipeline integration as a smoke-test gate:

```yaml
- task: AzureCLI@2
  displayName: 'Validate agent post-deploy'
  inputs:
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      RESPONSE=$(azd ai agent invoke \
        --name permit-inquiry-agent \
        --input "ping" 2>&1)
      if [ $? -ne 0 ]; then
        echo "Agent invocation failed: $RESPONSE"
        exit 1
      fi
      echo "Agent responded successfully"
      echo "$RESPONSE"
```

This kind of automated gate confirms that a deployed agent is reachable and responding before the pipeline completes. For teams that need to demonstrate deployment validation controls in audit reviews, terminal-based commands produce reproducible, loggable evidence of validation steps taken.

---

## Why This Matters for Government

Government IT organizations face a different risk profile than commercial enterprises when AI services fail. A private-sector chatbot going down is a customer experience problem. A government constituent services agent going down can mean residents miss benefit deadlines, cannot access vital records, or cannot reach agency information during emergencies.

Fast development and validation tools are especially valuable given the constraints government teams operate under:

**Change management windows.** Government systems often have narrow approved change windows. A development loop that lets teams catch problems locally - before opening a change request - reduces the number of change events and the risk of last-minute deployment failures during narrow maintenance windows.

**Smaller teams, broader scope.** Many state and local government IT departments maintain a large portfolio with a relatively small team. Tools that compress the testing and validation loop allow those teams to support more services at higher quality.

**Auditability.** Terminal-based workflows are inherently more auditable than portal clicks. Commands are logged, reproducible, and can be included in incident reports and deployment records as evidence of validation steps taken - supporting compliance with state IT governance requirements.

**IT governance and risk management alignment.** State and local IT governance frameworks - including those guided by NASCIO priorities and state CIO directives - increasingly emphasize risk-managed, measurable operations. Tooling that enables consistent pre-deployment testing and post-deployment validation supports the disciplined approach to IT operations that oversight bodies and constituents expect.

For agencies building or evaluating hosted AI agents on Azure AI Foundry, incorporating `azd ai agent run` and `azd ai agent invoke` into your standard development and deployment workflow from day one is a low-effort, high-return investment in reliability.

---

## Getting Started

1. **Install azd:** [https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/install-azd)
2. **Install the AI agents extension:** `azd extension install azure.ai.agents`
3. **Review the Foundry Agent Service overview:** [https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/overview)
4. **Follow the hosted agents quickstart:** [https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/quickstarts/quickstart-hosted-agent)
5. **Deploy your first agent:** [https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/extensions/azure-ai-foundry-extension](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/extensions/azure-ai-foundry-extension)

For questions, issues, or feature requests, the `azd` team is active on [GitHub](https://github.com/Azure/azure-dev/releases).

---

## Summary

The new `azd ai agent run` and `azd ai agent invoke` commands are a focused, practical addition to the developer toolchain for anyone building hosted AI agents on Azure AI Foundry. For government development teams operating under change management constraints, limited team bandwidth, and high public accountability, the ability to test an agent locally before deployment and validate it immediately afterward - from a single terminal session - is not a convenience. It is an operational discipline that reduces deployment risk and shortens recovery loops.

Faster validation at every stage of the deployment pipeline translates to more reliable constituent-facing services. That connection is exactly why these tools deserve a place in every government AI developer's workflow.
