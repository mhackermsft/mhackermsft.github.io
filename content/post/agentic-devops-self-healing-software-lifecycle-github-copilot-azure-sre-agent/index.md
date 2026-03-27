---
title: 'Agentic DevOps: Building a Self-Healing Software Lifecycle with GitHub Copilot and Azure SRE Agent'
date: 2026-03-27T18:20:55+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- How To
categories:
- AI
summary: Explore how AI agents can now handle the full software development lifecycle, from writing code to detecting and fixing production incidents autonomously, using GitHub Copilot and Azure SRE Agent.
draft: false
image_prompt: A massive autonomous clockwork mechanism of interlocking bronze and steel gears suspended in mid-air, with a glowing golden thread weaving through them in a continuous loop. A robotic hand on one side carefully places a new gear into the mechanism while a magnifying glass on the other side inspects the teeth of an existing gear for defects. Dramatic volumetric lighting from above casts deep shadows, with teal and navy blue ambient light filling the background. Sparks of energy pulse along the golden thread where it connects each gear, symbolizing the continuous self-healing cycle. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
audio: audio.mp3
---

The software industry has embraced AI-assisted development over the past few years. GitHub Copilot helps developers write code faster, generate tests, and scaffold entire application modules. But most organizations still treat the **operations side** of the software lifecycle as a manual, human-driven process. When a production incident fires at 3 AM, an engineer still wakes up, opens five browser tabs, and starts triaging.

What if AI agents could close that loop entirely? What if the same AI that helped write the code could also detect when it breaks, diagnose the root cause, implement the fix, and open a pull request for human review?

That is the promise of **Agentic DevOps**: a fully AI-augmented software development lifecycle where intelligent agents handle everything from code generation to incident response. A compelling open-source reference implementation called the [Agentic DevOps Demo](https://yortch.github.io/agentic-devops-demo/) showcases exactly this pattern using GitHub Copilot and Azure SRE Agent. Let's break down how it works and why government IT leaders should pay attention.

## What Is Agentic DevOps?

Agentic DevOps extends the DevOps philosophy of continuous integration and continuous delivery by introducing **AI agents at every stage** of the software lifecycle. Rather than AI serving as a passive autocomplete tool, agents operate autonomously: they build, deploy, monitor, diagnose, and remediate applications with minimal human intervention.

The key distinction is the **closed feedback loop**. Traditional DevOps pipelines are linear: code flows from development through CI/CD to production. When something breaks, a human investigates and manually authors a fix. In Agentic DevOps, AI agents monitor production, detect anomalies, perform root cause analysis, generate fixes, and route those fixes back through the same CI/CD pipeline. Humans remain in the loop for review and approval, but the cognitive burden of diagnosis and remediation shifts to the agents.

## The AI Agent Trio: GitHub Copilot Across the SDLC

The reference demo uses three complementary GitHub Copilot capabilities that together cover the full development lifecycle:

### 1. GitHub Copilot (IDE Assistant)

The foundation layer. Copilot's inline completions and chat-driven suggestions power every file in the project, from Spring Boot JPA entities and React components to Terraform infrastructure-as-code modules and GitHub Actions CI/CD workflows. In the demo, Copilot scaffolded all business entities with relationships, generated JUnit tests with a 100% pass rate, produced Playwright end-to-end test specs, and created the application's Material-UI theme. For government development teams that are often resource-constrained, this level of AI-assisted productivity means smaller teams can deliver more robust applications faster.

### 2. GitHub Copilot in the CLI (Terminal Intelligence)

Beyond the editor, [GitHub Copilot in the CLI](https://docs.github.com/en/copilot/github-copilot-in-the-cli) serves as the operational backbone. In the demo, it designed Terraform modules for Azure Container Apps and Azure Container Registry, authored the Azure Developer CLI (`azd`) project configuration with preprovision hooks, and built the Bicep infrastructure for Azure SRE Agent deployment. This is particularly valuable for infrastructure and platform teams who spend significant time writing deployment scripts and troubleshooting cloud configurations.

### 3. Copilot Coding Agent (Autonomous Developer)

The most transformative piece is the [Copilot Coding Agent](https://github.blog/news-insights/product-news/github-copilot-meet-the-new-coding-agent/). When a GitHub Issue is assigned to `@copilot`, the agent autonomously spins up a secure cloud sandbox powered by GitHub Actions, clones the repository, analyzes the codebase using retrieval-augmented generation (RAG) with GitHub code search, implements a targeted fix, and opens a draft pull request. The agent pushes commits incrementally so reviewers can track its reasoning. Existing repository policies like branch protections and required reviews still apply, and CI/CD workflows require human approval before running. This agent is available to [Copilot Enterprise and Copilot Pro+](https://docs.github.com/en/copilot) customers.

## Azure SRE Agent: The Operations Brain That Never Sleeps

[Azure SRE Agent](https://learn.microsoft.com/en-us/azure/sre-agent/overview) is an AI-powered site reliability engineering service that automates operational work: incident detection, investigation, root cause analysis, and remediation routing. Unlike static runbooks or simple alert scripts, SRE Agent **learns continuously** from every investigation it performs. It captures root causes, resolution steps, and team patterns to build institutional knowledge that persists and grows over time.

The service integrates with your existing operational ecosystem:

- **Monitoring**: Azure Monitor, Application Insights, Log Analytics, Grafana
- **Incident management**: Azure Monitor Alerts, PagerDuty, ServiceNow
- **Source control and CI/CD**: GitHub repositories and issues, Azure DevOps
- **Data sources**: Azure Data Explorer (Kusto) clusters, Model Context Protocol (MCP) servers

SRE Agent can manage all Azure services through the Azure CLI and REST APIs, covering compute (Container Apps, AKS, Functions), storage, networking, databases, and monitoring resources. You extend its capabilities through [custom subagents](https://learn.microsoft.com/en-us/azure/sre-agent/sub-agents) that package domain expertise, specialized tools, and knowledge bases for reuse.

### The Subagent Architecture

In the demo, two specialized subagents handle distinct responsibilities:

- **Incident Handler**: Triages incoming alerts, coordinates the investigation, creates structured GitHub Issues with incident summaries, affected metrics, error traces, root cause, and fix recommendations, then assigns the issue directly to `@copilot`.
- **Code Analyzer**: Performs deep code inspection by querying KQL (Kusto Query Language) logs, reading container metrics, and cross-referencing the GitHub repository source code to pinpoint the exact file and line that caused the incident.

Both subagents are grounded by [knowledge base documents](https://learn.microsoft.com/en-us/azure/sre-agent/sub-agents#knowledge-base-management) containing HTTP error runbooks and application architecture references, ensuring investigations follow established procedures.

## The Self-Healing Loop in Action

The crown jewel of the Agentic DevOps pattern is the **closed-loop incident response** where no human needs to write a single line of code to restore production. Here is how the loop operates:

1. **Proactive monitoring**: Azure SRE Agent runs three scheduled tasks continuously: health checks every 30 minutes, configuration drift detection every 6 hours, and a daily reliability report at 8 AM UTC. No alert is required to trigger an investigation.

2. **Alert detection**: Azure Monitor alert rules fire when anomalies occur, such as HTTP 5xx error spikes (severity 2), container restarts or out-of-memory kills (severity 1), and high response times (severity 3). Each alert routes to the SRE Agent's incident-handler subagent.

3. **Root cause analysis**: The code-analyzer subagent queries `ContainerAppConsoleLogs` via KQL, reads container metrics, and cross-references the GitHub repository to identify the exact commit and code change that caused the degradation. Average time to root cause in the demo: **under 2 minutes**.

4. **Automated issue creation**: The incident-handler creates a structured GitHub Issue with full context: incident timeline, affected metrics, error log excerpts, root cause identification (file and line number), and a concrete fix recommendation. The issue is labeled `sre-agent-detected` and assigned to `@copilot`.

5. **Autonomous fix implementation**: Copilot Coding Agent picks up the assigned issue, performs its own independent code analysis to validate the root cause, implements the minimal targeted fix, and opens a pull request referencing the incident issue.

6. **Human review and merge**: An engineer reviews the Copilot-authored PR. CI runs, tests pass, and the engineer merges. This is the **only human step** in the entire incident response pipeline.

7. **Production restored**: CD deploys the fix. The SRE Agent's next health check confirms all metrics are normal, closes the incident, and logs a recovery summary.

## Chaos Engineering with GitHub Agentic Workflows

The demo also showcases a novel approach to chaos engineering. Using [GitHub Agentic Workflows](https://github.com/github/gh-aw), natural-language markdown prompt files are compiled into locked workflow files (`.lock.yml`) that execute in sandboxed containers with egress firewalls. Only `api.github.com` and `api.githubcopilot.com` are permitted as outbound destinations.

The chaos engineering workflow autonomously selects from 13 realistic fault scenarios, modifies target files to introduce a breaking change, and opens a PR with a plausible commit message. Once merged, CI/CD deploys the broken code to Azure Container Apps, and the self-healing loop activates. This approach lets teams continuously validate their incident response capabilities without manual test construction.

## Why This Matters for Government

Government IT organizations face unique operational challenges that make the Agentic DevOps pattern particularly relevant:

**Staffing constraints and institutional knowledge loss.** Government agencies frequently struggle to recruit and retain experienced site reliability engineers. When senior staff leave, critical operational knowledge walks out the door. Azure SRE Agent's [memory and knowledge system](https://learn.microsoft.com/en-us/azure/sre-agent/memory) captures root causes, resolution steps, and team patterns automatically. New team members ramp up faster because the agent already knows deployment patterns, past incidents, and team procedures.

**24/7 availability with limited on-call staff.** Citizen-facing services like permitting portals, payment systems, and public health dashboards require high availability. With the self-healing loop, production incidents can be detected, diagnosed, and have a fix PR ready for review before a human even sees the alert. Mean time to resolution (MTTR) drops dramatically.

**Compliance and auditability.** Every step of the self-healing loop produces a traceable artifact: Azure Monitor alerts, SRE Agent investigation logs, structured GitHub Issues with full root cause analysis, and Copilot-authored PRs with commit history. This creates the audit trail that government compliance frameworks require.

**Doing more with less.** Smaller development teams can maintain larger application portfolios when AI agents handle routine operational tasks. The Copilot Coding Agent's security model, including branch protections, required human reviews, and gated CI/CD execution, ensures that autonomous fixes still go through proper change management.

### Current Availability Considerations

Government IT leaders should be aware of current platform availability as they plan adoption:

- **Azure SRE Agent** is currently available in Azure commercial regions: East US 2, Sweden Central, and Australia East. It is not yet available in Azure Government regions. Organizations using Azure commercial subscriptions can begin evaluating the service today. For the latest region availability, check the [supported regions documentation](https://learn.microsoft.com/en-us/azure/sre-agent/supported-regions).
- **GitHub Copilot Coding Agent** is available to Copilot Enterprise and Copilot Pro+ subscribers. Organizations should verify their GitHub Enterprise plan includes the necessary Copilot tier.
- **GitHub Copilot in the CLI** is available as part of the standard GitHub Copilot subscription.

## Getting Started

If you want to explore the Agentic DevOps pattern hands-on, the [open-source demo repository](https://yortch.github.io/agentic-devops-demo/) provides a complete reference implementation. The setup requires just three steps:

1. Authenticate with Azure and the Azure Developer CLI (`az login && azd auth login`)
2. Deploy the application infrastructure (`azd up`)
3. Deploy the SRE Agent configuration (`cd sre && azd up`)

For teams not ready to deploy the full demo, start by exploring these foundational capabilities:

- [Azure SRE Agent documentation](https://learn.microsoft.com/en-us/azure/sre-agent/overview) for understanding the monitoring and incident response automation
- [GitHub Copilot Coding Agent](https://github.blog/news-insights/product-news/github-copilot-meet-the-new-coding-agent/) for understanding autonomous code generation
- [Azure SRE Agent incident response](https://learn.microsoft.com/en-us/azure/sre-agent/incident-response) for setting up automated incident handling

## The Road Ahead

Agentic DevOps represents a fundamental shift in how software is built and operated. The pattern demonstrated here, with AI agents handling code generation, infrastructure provisioning, production monitoring, incident diagnosis, and automated remediation, is not a future vision. These are production-ready services available today on GitHub and Azure.

For government organizations striving to modernize their application portfolios while managing tight budgets and staffing constraints, the self-healing software lifecycle offers a compelling path forward. The human role evolves from firefighting to oversight: reviewing AI-generated fixes, approving deployments, and focusing on the high-value architectural and policy decisions that require human judgment.

The agents handle the rest.
