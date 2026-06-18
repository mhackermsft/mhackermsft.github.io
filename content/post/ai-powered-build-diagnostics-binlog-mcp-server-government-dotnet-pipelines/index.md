---
title: 'AI-Powered Build Diagnostics: Putting the Microsoft Binlog MCP Server to Work on Government .NET Pipelines'
date: 2026-06-18T14:45:18+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- How To
categories:
- App Modernization
summary: A hands-on look at Microsoft's preview Binlog MCP Server and how government .NET teams can use AI-assisted MSBuild binary log analysis to diagnose failures, slow builds, and dependency drift.
draft: false
image_prompt: An AI assistant analyzing MSBuild binary log diagnostics for a secure government .NET CI/CD pipeline, with abstract build graphs, timing traces, and Azure Government-inspired blue security visuals.
image: cover.png
audio: audio.mp3
---

Every government .NET team has lived this moment: a CI/CD pipeline that was green yesterday turns red today, the build log scrolls past for thousands of lines, and someone spends an afternoon searching MSBuild output for the target that broke. Sometimes the build still succeeds, but incremental builds have crept from 90 seconds to nine minutes and nobody can say why. For agencies running modernization programs on tight staffing and tight timelines, that lost time is real money and real schedule risk.

MSBuild already captures much of what you need in its binary log, or `.binlog`, format. The challenge is navigation. On June 17, 2026, the .NET team introduced the Microsoft Binlog MCP Server, a preview Model Context Protocol server that lets an AI assistant such as GitHub Copilot inspect MSBuild binary logs through structured tools. Instead of asking a developer to manually expand a build tree node by node, the assistant can query the log, summarize the failure, identify slow targets, and compare two builds in a conversation.

## What a binary log actually contains

If your team only keeps console output, you are leaving most of MSBuild's diagnostic value on the table. A binary log is a compressed, structured record of the build: property evaluation, target execution, task invocation, imported `.props` and `.targets` files, errors, warnings, and timing information. Binary logs can also include project files and other source files captured during the build.

For .NET CLI builds, the current command reference documents the binary logger as `-bl` or `--binaryLogger:<FILE>`. Generate the default `msbuild.binlog` like this:

```bash
dotnet build -bl
```

Or name the file explicitly:

```bash
dotnet build -bl:build-a.binlog
```

If you invoke `MSBuild.exe` directly, Microsoft Learn documents the binary logger switch as `-binaryLogger` with the short form `-bl`:

```bash
msbuild MyProject.proj -bl:build-a.binlog
```

Historically, you would open that file in the MSBuild Structured Log Viewer desktop app and inspect the tree by hand. That still works well, but it requires a human to know where to look.

## Enter the Binlog MCP Server

The Model Context Protocol is an open protocol for connecting AI applications to external tools and data sources. In this case, the external tool is an MSBuild binary log analyzer. The .NET Blog announcement describes the Microsoft Binlog MCP Server as a way for an AI assistant to investigate failures, trace property origins, analyze performance bottlenecks, compare builds, and read embedded source files.

The exact tool inventory is moving quickly. The June 17 announcement highlights the core tool set, while the current NuGet package documentation lists more than 30 tools. For practical use, think in terms of capabilities rather than a fixed count:

**Build investigation:** `binlog_overview`, `binlog_errors`, `binlog_warnings`, `binlog_search`, `binlog_projects`, `binlog_properties`, `binlog_items`, `binlog_imports`, and `binlog_explain_property` help the assistant move from a high-level build summary to the target, task, file, line, or property assignment that matters.

**Embedded files:** `binlog_files` and `binlog_search_files` let the assistant inspect files embedded in the binary log, such as project files, props files, targets files, and related build inputs.

**Performance analysis:** `binlog_expensive_projects`, `binlog_expensive_targets`, and `binlog_expensive_tasks` rank the slowest parts of the build. Current package documentation also lists specialized performance tools such as analyzer and target timing analysis.

**Build comparison:** `binlog_compare` compares two binlogs across properties, packages, and related build data.

The search tool uses the Structured Log Viewer search syntax. That includes node filters such as `$error`, `$warning`, `$task`, `$target`, and `$project`, plus scoped expressions such as `under(...)` and `project(...)`.

## Wiring it into your environment

The supported adoption path is the .NET Agent Skills repository. Its `dotnet-msbuild` plugin bundles MSBuild build skills, agents, and the Binlog MCP server configuration.

**Visual Studio:** Visual Studio MCP documentation lists Visual Studio 2026 or Visual Studio 2022 version 17.14 with current servicing updates as prerequisites for MCP server support. The .NET Blog announcement says that, after the `dotnet-msbuild` plugin is installed, Copilot Chat in agent mode can discover the Binlog MCP Server and make the `binlog_*` tools available for conversations about a `.binlog` file in the solution.

**Visual Studio Code:** VS Code plugin support is documented as a preview capability in the `dotnet/skills` repository. Enable plugin support and add the marketplace in `settings.json`:

```json
{
  "chat.plugins.enabled": true,
  "chat.plugins.marketplaces": ["dotnet/skills"]
}
```

Then use Copilot Chat's plugin experience to browse and install the `dotnet-msbuild` plugin.

**Command line:** For terminal-based assistants that support the plugin marketplace, the `dotnet/skills` README documents this flow:

```text
/plugin marketplace add dotnet/skills
/plugin install dotnet-msbuild@dotnet-agent-skills
```

Restart the assistant after installation and verify loaded skills with:

```text
/skills
```

Avoid treating the NuGet package as a stable direct-use interface unless you have a specific reason to do so. The current NuGet package page says the tool is meant to be used by plugins distributed through `dotnet/skills` and that Microsoft does not guarantee compatibility or support for direct usage.

## A diagnosis walkthrough

Start by capturing the log:

```bash
dotnet build -bl
```

Then ask the assistant: 'My build failed. Investigate msbuild.binlog and tell me what went wrong.'

Behind the scenes, the assistant can call `binlog_overview` for the high-level status, `binlog_errors` for the actual errors with project, target, task, file, and line context, and `binlog_explain_property` or `binlog_search` to trace a root cause. A common example is a `TargetFramework`, `Configuration`, or NuGet feed property that resolved differently on the build agent than it did on a developer machine.

The performance workflow is similar. Point the assistant at a slow build and ask what changed. It can use the expensive-project, expensive-target, and expensive-task tools to identify the slowest work. That can surface issues such as a code generation target re-running on every incremental build because an input timestamp keeps changing.

## Catching dependency and configuration drift

Comparison is especially useful for government teams that need reproducible build evidence for pull requests, change records, and release reviews. Capture two logs from known points:

```bash
git checkout main
dotnet build -bl:build-a.binlog

git checkout my-feature-branch
dotnet build -bl:build-b.binlog
```

Then ask: 'Compare build-a.binlog and build-b.binlog. What MSBuild properties and package versions changed, and did any of those changes affect build performance?'

The assistant can use `binlog_compare` to diff properties and packages, then correlate differences with timing data. That turns a side-by-side review into a documented investigation that a developer can paste into a pull request, release note, or incident record.

## A note on telemetry and controlled environments

The Binlog MCP Server's current package documentation says telemetry is on by default and can be disabled with the standard .NET CLI telemetry opt-out variable. For Bash:

```bash
export DOTNET_CLI_TELEMETRY_OPTOUT=1
```

For Windows PowerShell:

```powershell
$env:DOTNET_CLI_TELEMETRY_OPTOUT = "1"
```

The .NET Blog announcement states that no binlog content, file paths, or raw error messages are collected, and that filenames are HMAC-SHA256 hashed for correlation. The NuGet package documentation adds that setting `DOTNET_CLI_TELEMETRY_OPTOUT=1` disables telemetry destinations.

For public-sector environments, remember that this Binlog MCP Server runs as a local tool, while MCP as a standard also supports remote servers. Either way, an MCP server can expose data to the host AI application. The MCP specification and VS Code documentation emphasize user consent, tool approval, and trust review. Treat any MCP server as part of your development security boundary: install it from trusted sources, use organization policy where available, and align tool usage with your agency's data-handling rules.

## Why This Matters for Government

State and local IT organizations are modernizing legacy line-of-business systems on .NET while competing for scarce senior engineering time. Faster build diagnosis protects delivery schedules, reduces triage cost, and improves the evidence trail around software changes.

Three points make this practical rather than aspirational:

1. **It works with MSBuild data you can already produce.** Binary logs are a documented MSBuild feature, and `-bl` drops into existing build steps without a new build platform.
2. **It supports controlled diagnostic workflows.** The server analyzes local `.binlog` files through MCP tools, telemetry can be disabled with `DOTNET_CLI_TELEMETRY_OPTOUT`, and teams can decide which assistants, plugins, and MCP servers are approved.
3. **It helps preserve institutional knowledge.** When build investigation is captured as a repeatable conversation over a binary log, short-tenured teams and contractors can understand why a build failed without relying on one senior engineer's memory.

This fits common public-sector concerns such as FedRAMP-aligned cloud governance, CJIS-sensitive development practices, IRS Publication 1075 workloads, HIPAA-regulated systems, modernization budgets, and auditability. Microsoft compliance documentation notes that Azure Government is built for US government agencies and partners with government security and compliance requirements, and that Microsoft cloud services have documented FedRAMP and CJIS compliance offerings. The Binlog MCP Server does not replace agency compliance review, but it can make the software delivery process more traceable and easier to govern.

A reasonable adoption path is straightforward: enable binary logs for CI builds, archive `.binlog` files as protected pipeline artifacts, install the `dotnet-msbuild` plugin for approved developer environments, set `DOTNET_CLI_TELEMETRY_OPTOUT=1` in standard agent images when required, and use the assistant for first-pass triage on failed or slow builds.

## Try it and give feedback

The .NET team describes the Microsoft Binlog MCP Server as preview and is inviting feedback in the `dotnet/skills` repository. If your modernization program runs on .NET and your developers already use AI coding assistants, this is a practical place to start: generate a binary log, ask the assistant what happened, and use the result to shorten build triage while keeping engineers focused on shipping.

### Sources

- [AI-Powered MSBuild Investigation with the Microsoft Binlog MCP Server](https://devblogs.microsoft.com/dotnet/msbuild-binlog-mcp-server/) - .NET Blog, June 17, 2026
- [Obtain build logs with MSBuild](https://learn.microsoft.com/en-us/visualstudio/msbuild/obtaining-build-logs-with-msbuild) - Microsoft Learn
- [MSBuild command-line reference](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-command-line-reference) - Microsoft Learn
- [dotnet build](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-build) - Microsoft Learn
- [Use MCP servers in Visual Studio](https://learn.microsoft.com/en-us/visualstudio/ide/mcp-servers?view=vs-2022) - Microsoft Learn
- [Add and manage MCP servers in VS Code](https://code.visualstudio.com/docs/copilot/chat/mcp-servers) - Visual Studio Code documentation
- [.NET SDK and .NET CLI telemetry](https://learn.microsoft.com/en-us/dotnet/core/tools/telemetry) - Microsoft Learn
- [Microsoft.AITools.BinlogMcp](https://www.nuget.org/packages/Microsoft.AITools.BinlogMcp) - NuGet Gallery
- [dotnet/skills](https://github.com/dotnet/skills) - GitHub
- [MSBuild Structured Log search syntax](https://msbuildlog.com/syntax)
- [Model Context Protocol specification](https://modelcontextprotocol.io/specification/2025-06-18)
- [Azure Government overview](https://learn.microsoft.com/en-us/azure/azure-government/documentation-government-welcome)
- [Microsoft FedRAMP compliance offering](https://learn.microsoft.com/en-us/compliance/regulatory/offering-fedramp)
- [Microsoft CJIS compliance offering](https://learn.microsoft.com/en-us/compliance/regulatory/offering-cjis)
