---
title: 'Upgrade Legacy .NET Apps from Anywhere: The GitHub Copilot modernize-dotnet Agent'
date: 2026-03-16T16:54:59+00:00
author: Mike Hacker
tags:
- App Modernization
- AI
- How To
- Announcements
categories:
- App Modernization
summary: The newly announced GitHub Copilot modernize-dotnet agent brings AI-assisted .NET modernization to Visual Studio, VS Code, the CLI, and GitHub.com - giving government IT teams a traceable, developer-friendly path to eliminate technical debt across their .NET application portfolios without a full rewrite.
draft: false
image_prompt: A dramatic close-up of a robotic hand carefully lifting an old rusty gear out of a complex mechanical assembly and replacing it with a gleaming new chrome gear, sparks of blue energy arcing between the old and new parts, dark industrial background with warm amber side lighting and cool blue fill light. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
---

Government IT departments are sitting on a growing backlog of legacy .NET applications. Permitting systems built on ASP.NET WebForms. Case management tools targeting .NET Framework 4.5. Internal service APIs that predate .NET Core entirely. These applications are not broken - they still run critical workflows every day - but they carry compounding costs: end-of-life runtimes accumulate unpatched vulnerabilities, aging frameworks block adoption of modern cloud services, and every year that passes makes the eventual upgrade harder and riskier.

The traditional response to this problem is a rewrite, a contractor engagement, or a modernization project that stretches across multiple budget cycles. None of those options scale to the size of the problem. What changes the equation is an AI agent that can assess your codebase, produce a detailed upgrade plan, and execute code transformations automatically - in the development environment your team already uses.

Microsoft announced exactly that capability on March 12, 2026, with the expanded release of the [GitHub Copilot modernize-dotnet agent](https://devblogs.microsoft.com/dotnet/modernize-dotnet-anywhere-with-ghcp/). The same AI-powered modernization workflow that was previously limited to Visual Studio can now run across Visual Studio, Visual Studio Code, the GitHub Copilot CLI, and directly on GitHub.com - wherever your developers already work.

## What the modernize-dotnet Agent Does

The modernize-dotnet agent is built on GitHub Copilot's agentic platform and follows a structured three-stage workflow: **assess, plan, execute**. Every stage produces a Markdown artifact committed to your repository under `.github/upgrades`, making the entire process transparent, reviewable, and version-controlled.

### Stage 1 - Assessment

The agent begins by examining your project structure, source code, configuration files, and dependencies. It produces an `assessment.md` document that identifies breaking changes, API compatibility issues, deprecated patterns, and the full scope of what needs to change before the upgrade is complete. Rather than discovering problems after you attempt the upgrade, you get a complete picture of the work before a single line of code is modified.

For government IT teams maintaining applications that may have been built by contractors who are no longer engaged, this assessment capability is particularly valuable. AppCAT-style discovery surfaced through a Copilot interface means your current developers can get an accurate inventory of what a legacy application actually uses - even when internal documentation is sparse or absent.

### Stage 2 - Planning

The agent converts the assessment into a `plan.md` file: a detailed specification that explains exactly how to resolve every identified problem. This includes upgrade strategies, dependency upgrade paths, refactoring approaches for deprecated APIs, and risk mitigations for high-impact changes.

Critically, you can edit this file before execution begins. If your organization has specific architectural standards - particular logging frameworks, preferred Azure service mappings, internal security controls - you can annotate the plan, and the agent will incorporate your guidance when it executes. This is not a black-box upgrade process; it is a collaborative proposal that your team reviews and approves.

### Stage 3 - Execution

The agent breaks the plan into a `tasks.md` file: sequential, concrete tasks with individual validation criteria. When you tell the agent to proceed, it begins executing those tasks one by one, committing changes to a Git branch at each step. If a task fails - a build error, a test regression, a dependency conflict - the agent attempts to diagnose and fix the problem automatically. If it cannot resolve the issue on its own, it pauses and asks for your help. When you provide a correction, the agent learns from that intervention and applies the same fix if the same issue appears elsewhere in the codebase.

## What the Agent Can Upgrade

The [modernize-dotnet agent](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization/overview) supports a broad range of .NET project types:

- **ASP.NET Core** applications - including MVC, Razor Pages, and Web API
- **Blazor** applications
- **Azure Functions**
- **Windows Presentation Foundation (WPF)**
- **Windows Forms**
- **Class libraries and console applications**
- **Test projects** - MSTest, NUnit, and others

Supported upgrade paths include moving from older .NET versions to the latest release, upgrading from .NET Framework to modern .NET, and migrating application components to Azure services. Azure-targeted migrations cover a practical set of common scenarios:

- **Managed Identity-based database connections** to Azure SQL DB, Azure SQL Managed Instance, or Azure PostgreSQL - replacing legacy username/password authentication
- **Azure Blob Storage and Azure File Storage** migration from local file system I/O
- **Microsoft Entra ID** migration from Windows Active Directory for authentication and authorization
- **Azure Key Vault** integration to replace plaintext credentials in configuration files
- **Azure Service Bus** migration from MSMQ or RabbitMQ
- **Azure Cache for Redis** with Managed Identity, replacing in-process or standalone Redis implementations
- **OpenTelemetry on Azure** migration from log4net, Serilog, or Windows Event Log

For government applications, several of these migration scenarios map directly to Zero Trust architecture requirements - particularly the Managed Identity database connection migration and the Azure Key Vault credential management pattern.

## Run It From Wherever You Already Work

One of the most significant aspects of this release is that the modernize-dotnet agent is no longer tied to a single IDE. Government IT teams vary widely in their tooling - some developers prefer Visual Studio, others work in VS Code or primarily from the terminal.

**Visual Studio**: Right-click any solution or project in Solution Explorer and select **Modernize**, or type `@Modernize` in the GitHub Copilot Chat window. This is the fully integrated experience, with real-time output visible in the Output window.

**Visual Studio Code**: Open Copilot Chat and type `@modernize-dotnet` followed by your upgrade request. This works cross-platform - Linux and macOS developers on your team can participate in the modernization workflow without needing Windows. See the [installation guide](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization/install) for VS Code setup instructions.

**GitHub Copilot CLI**: For terminal-first engineers, install the plugin with two commands:
```
/plugin marketplace add dotnet/modernize-dotnet
/plugin install modernize-dotnet@modernize-dotnet-plugins
```
Then select the agent with `/agent` and prompt it directly: `upgrade my solution to a new version of .NET`. The agent generates assessment, plan, and task artifacts in the repository and executes the upgrade without leaving the shell.

**GitHub.com**: For teams that want modernization to become a collaborative, pull-request-driven workflow, the agent can be added as a custom coding agent to a GitHub repository. Generated upgrade artifacts live alongside code in the repository, and the entire modernization process becomes part of your standard code review workflow rather than something that happens on an individual developer's machine.

> **Prerequisites**: A GitHub Copilot subscription is required - Copilot Free (in Visual Studio 2026 version 18.1+), Copilot Pro, Pro+, Business, or Enterprise all qualify. .NET Framework migrations require Windows. The [installation guide](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization/install) covers setup for each environment.

## Custom Skills: Encoding Your Organization's Standards

Government IT organizations do not operate in a vacuum. You likely have internal architectural standards, approved cloud service configurations, security control requirements, and coding conventions that every modernized application must follow. The modernize-dotnet agent supports **custom skills** that allow your organization to encode these standards directly into the modernization workflow.

Skills are reusable, opinionated behaviors that the agent applies consistently across every upgrade it performs in your repository. If your organization requires that all database connections use a specific connection string pattern, that all logging infrastructure target a specific Azure Monitor workspace, or that Managed Identity configuration follow an internal naming convention, those requirements can be encoded as skills and automatically applied - without relying on developers to remember them manually or on reviewers to catch omissions.

This is a meaningful capability for agencies under compliance frameworks that require consistent security control implementation across application portfolios. Consistency at scale is exactly the problem that compliance teams face and that individual developer judgment alone cannot reliably solve.

## Why This Matters for Government

Government IT organizations face a specific set of pressures that make the modernize-dotnet agent especially relevant.

**End-of-life runtimes are a compliance liability.** .NET Framework 4.x applications running on Windows Server 2012 R2 - which reached end of extended support in October 2023 - generate audit findings under FISMA, FedRAMP, and equivalent state security frameworks. Every application still targeting an end-of-life runtime is a line item on your Plan of Action and Milestones (POA&M). The modernize-dotnet agent provides a documented, traceable path to remediate those findings at scale.

**Zero Trust mandates require modern authentication.** CISA's Zero Trust Maturity Model, OMB M-22-09, and equivalent state-level directives all point in the same direction: eliminate shared credentials, adopt managed identities, and remove hardcoded secrets from application configuration. The predefined migration tasks in the modernize-dotnet agent - Managed Identity database authentication, Azure Key Vault for credentials - are direct implementations of these requirements. An agent that applies these patterns systematically across an entire codebase is more reliable than relying on each developer to make the right choices in every code path.

**Workforce constraints are not going away.** Government IT departments consistently report difficulty recruiting and retaining experienced developers. An AI agent that handles the mechanical work of dependency upgrades, API compatibility fixes, and authentication pattern migrations frees your developers to focus on application logic, business rules, and the judgment calls that genuinely require human expertise. The same small team can move through a larger application portfolio in a given budget cycle.

**The artifacts matter for oversight.** The three-stage workflow's Markdown outputs - `assessment.md`, `plan.md`, `tasks.md` - are more than operational artifacts. They are documentation that can be incorporated into security assessment packages, change management records, and program reporting to oversight bodies. The Git commit trail that the agent creates at each step provides an immutable record of what changed, when, and why - which is exactly what auditors and security reviewers need to validate that changes were deliberate and reviewed.

**Portability across Azure commercial and Azure Government.** The modernize-dotnet agent runs as a local IDE extension or CLI tool. Analysis and code transformation happen in your development environment, not in a cloud service. This means agencies with workloads in Azure US Government regions, on M365 GCC tenants, or in restricted network environments can use the modernization tooling without routing source code through commercial cloud endpoints. The Azure services that applications are being migrated *to* - Azure SQL, Azure Key Vault, Azure Service Bus - are all available in Azure US Government regions.

## Getting Started: A Practical First Step

If your agency is ready to begin, the most effective starting point is a pilot on a single representative application - something important enough to be worth modernizing, but not so critical that any disruption to the process is unacceptable.

1. **Install the agent** in your preferred development environment using the [installation guide](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization/install).
2. **Open the target project** in Visual Studio or VS Code and invoke the agent with `@Modernize` or `@modernize-dotnet`.
3. **Review the assessment**: Before approving any code changes, read through `assessment.md` carefully. This is where you learn what the agent found and validate that its understanding of your application is accurate.
4. **Adjust the plan**: Edit `plan.md` if you have organizational requirements that the agent did not automatically capture. This is your opportunity to inject your internal standards before execution begins.
5. **Execute with oversight**: Run the execution stage and monitor progress through `tasks.md`. Review each Git commit before it is merged to your main branch.
6. **Establish a repeatable process**: Once your team has run the agent on one application and understands the workflow, build a standard procedure that other development teams can follow across the rest of your portfolio.

The [GitHub Copilot modernization FAQ](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization/faq) covers common questions about what data is collected, how to disable telemetry, and the limitations of the agent - worth reviewing before your first production use.

## The Window to Act Is Open

The modernize-dotnet agent does not eliminate the work of modernizing a legacy .NET portfolio - there will always be edge cases, business logic that requires human interpretation, and architectural decisions that no AI can make for you. What it eliminates is the mechanical scaffolding work that currently consumes the majority of developer time in any modernization effort: dependency mapping, compatibility analysis, boilerplate code transformation, build error remediation.

For government IT leaders managing application portfolios accumulated over decades, that shift in where human effort is required is the difference between a modernization program that makes steady, measurable progress and one that stalls at the assessment phase indefinitely. The tools are available now, across the development environments your teams already use. The technical debt is not getting smaller.

---

**Resources:**
- [Modernize .NET Anywhere with GitHub Copilot - .NET Blog (March 2026)](https://devblogs.microsoft.com/dotnet/modernize-dotnet-anywhere-with-ghcp/)
- [What is GitHub Copilot modernization - Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization/overview)
- [Install GitHub Copilot modernization](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization/install)
- [Upgrade a .NET app with GitHub Copilot modernization](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization/how-to-upgrade-with-github-copilot)
- [GitHub Copilot modernization FAQ](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization/faq)
- [Plan and perform .NET upgrades - Microsoft Learn](https://learn.microsoft.com/en-us/dotnet/core/porting/)
- [modernize-dotnet GitHub repository](https://github.com/dotnet/modernize-dotnet)
