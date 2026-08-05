---
title: 'GitHub Copilot Upgrade vs. the Modernization Agent: Choosing the Right Starting Point'
date: 2026-08-05T15:14:11+00:00
author: Mike Hacker
tags:
- App Modernization
- AI
- How To
categories:
- Azure
- GitHub Copilot
- App Modernization
summary: A source-grounded guide to when government developers should reach for GitHub Copilot upgrade versus the GitHub Copilot Modernization Agent, with verified quick starts and a decision table.
draft: false
image_prompt: 'A clean Microsoft Azure style illustration showing a government application portfolio branching into two modernization paths: GitHub Copilot upgrade for a single repository and Modernize CLI for portfolio planning, with reviewable Git artifacts and human oversight.'
image: cover.png
audio: audio.mp3
---

Earlier this year, we described AI agents as a modernization team and walked through the then-current `modernize-dotnet` naming inside GitHub Copilot. The product documentation has changed since then. This post updates the guidance so government teams are working from first-party sources current on August 5, 2026.

The most important correction is terminology: the repository-level .NET agent is documented as **GitHub Copilot upgrade**. It is also inaccurate to treat one agent as planning-only and the other as execution-only. Both experiences use human-in-the-loop workflows for assessment, planning, code changes, and validation. The right question is not which one is allowed to make changes. It is what problem you are starting from.

## Two agents, one modernization family

Microsoft groups these capabilities under GitHub Copilot modernization. Per the [modernization overview](https://learn.microsoft.com/en-us/azure/developer/github-copilot-app-modernization/overview), last updated June 12, 2026, the experience is delivered in two complementary layers:

- **GitHub Copilot upgrade** is the interactive, repository-level agent documented for .NET upgrade work. It helps upgrade .NET runtimes, frameworks, dependencies, and selected modernization patterns from a developer environment. In the broader GitHub Copilot modernization family, the IDE experience also includes language, framework, and toolset upgrades for Java and C++.
- **The GitHub Copilot Modernization Agent**, delivered through the **Modernize CLI**, is built for architects and application owners who need to orchestrate work across multiple applications, define reusable organizational standards, plan Azure migration, and delegate work to local execution or cloud coding agents.

Both run agentic workflows. Neither is a deterministic one-click converter, and both require review before changes are merged.

## What changed since March

Several updates matter for teams returning to earlier notes:

- **The repository-level .NET agent is documented as GitHub Copilot upgrade.** The plugin README, install documentation, and agent picker use the Upgrade naming.
- **The Upgrade Dashboard is documented.** The [dashboard doc](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/dashboard), updated August 4, 2026, describes an eight-tab canvas: Overview, Scenario, Tasks, Projects, Dependencies, Assessment, Options, and Activity.
- **The .NET scenario and skills reference is broader than framework-only upgrades.** The [scenarios and skills reference](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/scenarios-and-skills) lists SDK-style conversion, Newtonsoft.Json to System.Text.Json, System.Data.SqlClient to Microsoft.Data.SqlClient, Azure Functions in-process to isolated worker, Semantic Kernel to Microsoft Agent Framework, Aspire integration, WebForms-to-Blazor, and 30+ built-in skills that auto-load based on detected code.
- **The Modernize CLI supports portfolio assessment.** The [batch assessment doc](https://learn.microsoft.com/en-us/azure/developer/github-copilot-app-modernization/modernization-agent/batch-assess) confirms assessment across Java, .NET, and JavaScript/TypeScript applications. It also describes codebase insights for architecture, API contracts, configuration, business workflows, dependencies, and data model.
- **The .NET FAQ routes Azure migration to the Modernization Agent.** The [FAQ](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/faq), updated August 4, 2026, states that Azure migration scenarios are handled by the GitHub Copilot modernization agent.

## Where to start

| Start with... | When you need to... | Host | Intended outcome |
|---|---|---|---|
| **GitHub Copilot upgrade** | Upgrade a single .NET repository's runtime, framework, dependencies, or supported modernization patterns | Visual Studio, Visual Studio Code, GitHub Copilot CLI, GitHub Copilot app, GitHub.com | A reviewable upgrade branch with assessment, plan, task artifacts, commits, and validation output |
| **Modernization Agent (Modernize CLI)** | Assess a portfolio, coordinate multiple repositories, define reusable organizational standards, plan Azure migration, delegate to cloud coding agents, or wire modernization into CI/CD | Modernize CLI / TUI | A portfolio assessment, migration plan, migration waves, and repeatable execution workflow |

Either agent can run independently, or you can chain them: assess the portfolio with the Modernization Agent, then hand individual repositories to GitHub Copilot upgrade. There is no mandatory handoff in either direction.

## Quick start: GitHub Copilot upgrade

In the GitHub Copilot CLI, add the marketplace, install the plugin, and pick the agent. These commands are verified against the [upgrade-agent-plugins README v1.1.290](https://raw.githubusercontent.com/microsoft/upgrade-agent-plugins/v1.1.290/README.md), the [v1.1.290 tag page](https://github.com/microsoft/upgrade-agent-plugins/releases/tag/v1.1.290), and the [install doc](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/install):

```text
/plugin marketplace add microsoft/upgrade-agent-plugins
/plugin install upgrade-agent@upgrade-agent-plugins
/agent
```

Select **Upgrade**, then prompt in natural language, for example: `upgrade my solution to .NET 10`. You do not need to memorize scenario names. The agent maps your intent to the right scenario and skills.

Per the [.NET upgrade overview](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-upgrade/overview), when you ask the agent to upgrade an app in a Git repository, Copilot prompts you to create a working branch, assesses the project, and runs a three-stage workflow: assessment, planning, and execution. The agent writes reviewable Markdown under `.github/upgrades/{scenarioId}/`, including `assessment.md`, `upgrade-options.md`, `plan.md`, `tasks.md`, `scenario-instructions.md`, and per-task files. Because state lives in that folder, you can close your IDE, switch sessions, or move between supported development environments and continue from repository state.

Copilot creates commits during the upgrade workflow, and the FAQ states that each task can be accepted selectively through standard Git operations. You can review history with `git log --oneline` and use `git cherry-pick` when you need to bring over specific commits.

### The Upgrade Dashboard

The dashboard is available through the **GitHub Copilot app for desktop** and the **GitHub Copilot CLI**. In the desktop app it opens as a side panel; in the CLI it opens in your browser. The overview states that the dashboard is not currently available in Visual Studio, Visual Studio Code, or other IDEs. Ask Copilot to `show the upgrade dashboard` during an active session to open it.

## Quick start: the Modernization Agent

Install the Modernize CLI and authenticate. These commands are verified against the [Modernize CLI README v1.0.74](https://raw.githubusercontent.com/microsoft/modernize-cli/v1.0.74/README.md), the [v1.0.74 release page](https://github.com/microsoft/modernize-cli/releases/tag/v1.0.74), the [quickstart](https://learn.microsoft.com/en-us/azure/developer/github-copilot-app-modernization/modernization-agent/quickstart), the [winget install reference](https://learn.microsoft.com/en-us/windows/package-manager/winget/install), and the [GitHub CLI auth reference](https://cli.github.com/manual/gh_auth_login):

```powershell
winget install GitHub.Copilot.modernization.agent
gh auth login
modernize
```

Running `modernize` opens the interactive TUI with Assess, Plan, Execute, and a quick-start Upgrade option. You can also drive documented commands non-interactively:

```bash
modernize assess
modernize upgrade ".NET 10"
modernize assess --source https://github.com/org/repo.git
modernize assess --source https://github.com/org/repo-one.git --source https://github.com/org/repo-two.git
modernize assess --source .github/modernize/repos.json
modernize assess --source .github/modernize/repos.json --delegate cloud --wait
modernize plan execute --plan-name modernization-plan --no-tty
modernize update
```

For portfolio assessment through a repository configuration file, use the interactive flow and select **From a config file**, or use the documented non-interactive form `modernize assess --source .github/modernize/repos.json`. The batch assessment documentation says the agent automatically detects `.github/modernize/repos.json` in interactive mode, lets you provide a custom config path, and also shows the config file path as a supported `--source` value for non-interactive assessment. The Modernize CLI README also documents repeatable `--source` values for direct directory paths or Git URLs.

A repository config entry includes a `name` plus either a Git `url` or local `path`, with optional `branch`. The full format can also include an `apps[]` section for grouped reporting. Assessment output lands in `.github/modernize/assessment/`, and plans are written to `.github/modernize/{plan-name}/plan.md`.

The CLI supports local execution or cloud coding agent delegation for parallel processing, headless operation through `--no-tty` for CI/CD, direct `modernize upgrade` runs, and a `modernize update` command that adapts to the install method.

One caveat on cloud delegation: the batch assessment documentation says it requires GitHub.com repository URLs. Local paths and non-GitHub providers, such as GitLab or Azure DevOps, must use local execution.

## Be precise about availability

The [modernization overview](https://learn.microsoft.com/en-us/azure/developer/github-copilot-app-modernization/overview) states the current status clearly:

- **Generally available:** IDE experience for language, framework, and toolset upgrades for .NET, Java, and C++.
- **Generally available:** IDE migration scenarios for .NET and Java.
- **Public preview:** Modernization Agent CLI experience for application assessment and planning.

The released CLI exposes execution and upgrade commands, but treat the Modernization Agent CLI experience as preview unless a newer first-party statement labels a specific capability generally available.

## Set realistic expectations

These are iterative agent workflows, not deterministic converters. The GitHub Copilot upgrade FAQ is candid: upgrade suggestions are not guaranteed to follow best practices, and code fixes or corrections you provide during the upgrade process are not remembered as training data for future upgrades. Repository state and confirmed decisions can be saved in files such as `scenario-instructions.md`, but those files guide the workflow in your repository; they do not train the underlying model.

Assessments, plans, tasks, branches, commits, builds, and tests remain reviewable, and every change must be reviewed and tested before you merge. GitHub Copilot upgrade also requires an internet connection and GitHub Copilot cloud infrastructure; per the FAQ, it cannot run offline.

## Why This Matters for Government

State and local IT organizations rarely modernize one app. They inherit portfolios: .NET Framework web apps, WCF services, in-process Azure Functions, Java systems, and front ends maintained by lean teams facing workforce constraints.

- **Portfolio governance and repeatable standards.** The Modernization Agent lets architects define reusable skills and apply them across repositories, so security and coding standards can be applied consistently rather than reinvented per team.
- **Auditability.** Both agents produce reviewable artifacts and Git commits. For agencies that need evidence for change control, procurement review, or compliance programs such as FedRAMP, StateRAMP, CJIS, IRS Publication 1075, or HIPAA, the `.github/upgrades/` and `.github/modernize/` folders can become part of the review packet. They are evidence aids, not compliance certifications.
- **Security review.** Upgrade skills cover cryptography namespace changes, ADAL-to-MSAL, and modern Azure SDKs. Batch assessment includes Java CVE scanning today, with .NET and JavaScript/TypeScript security scanning described as roadmap items in the batch assessment documentation.
- **Migration planning.** Batch assessment can turn a broad statement such as get off end-of-life frameworks into aggregated reports with recommended Azure services, target platforms, upgrade paths, migration strategies, and migration waves.
- **Human oversight.** Nothing should merge without review. That aligns with the change-control discipline government environments already require.

A practical note: keep feature-availability claims tight. This post does not make assertions about Azure Government availability, air-gapped use, data residency, or compliance certification for these agents, because the cited first-party docs do not establish those claims. If your workloads sit in a constrained cloud, validate feature parity before committing to a modernization plan.

## The bottom line

If a developer is holding one .NET repository that needs to move to .NET 10 or another supported target, start with **GitHub Copilot upgrade**. If an architect is holding a portfolio that needs assessment, reusable standards, Azure migration planning, cloud delegation, or CI/CD orchestration, start with the **Modernization Agent**. Both can support reviewable modernization work, and the two compose cleanly when a portfolio plan needs to become concrete repository upgrades.
