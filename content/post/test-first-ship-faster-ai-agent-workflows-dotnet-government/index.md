---
title: 'Test First, Ship Faster: AI Agent Workflows for Government .NET Development Teams'
date: 2026-03-20T12:41:06+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- How To
- Training
categories:
- App Modernization
summary: Learn how government .NET development teams can adopt the same AI agent workflow patterns used by the .NET MAUI team to cut issue resolution time by 50-70%, improve test coverage, and deliver better software with the staff they already have.
draft: false
image_prompt: A dramatic close-up of an intricate clockwork mechanism with dozens of interlocking bronze and silver gears of varying sizes, each tooth precisely aligned with the next, bathed in warm amber light from below and cool blue light from above. A luminous golden gear glows at the center of the mechanism, radiating light outward through the surrounding components. The background fades into deep shadow, emphasizing the precision and interdependence of each moving part. Sharp depth of field on the central gear, bokeh on the outer edges. Photorealistic, highly detailed machined surfaces with visible wear patterns and reflective metal sheen. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
---

The .NET MAUI team recently shared something remarkable: by adopting a structured, multi-agent AI workflow, contributors reduced issue resolution time by 50 to 70 percent while simultaneously improving test coverage and code quality. The result wasn't achieved by buying new tools or adding headcount - it came from applying engineering discipline to AI-assisted development through a repeatable, test-first workflow pattern.

For government development teams maintaining citizen-facing portals, permitting systems, or internal line-of-business applications built on .NET, this same pattern is directly applicable today. The tools exist, the workflow is proven, and the productivity gains are measurable.

## The Challenge Facing Government Development Teams

Government software shops face constraints that compound over time. Hiring freezes limit team growth. Procurement cycles add months to tooling decisions. Senior developers who hold institutional knowledge eventually leave, and their expertise walks out the door with them. Meanwhile, the backlog of open defects and feature requests grows.

The result is a familiar pattern: every bug takes longer to fix than it should, test coverage is inconsistent across codebases, and code reviews are bottlenecked on the two or three people who truly understand the system. Junior developers produce fixes that inadvertently introduce regressions. The same classes of defects appear repeatedly.

AI coding agents address exactly this problem - not by replacing developers, but by giving every developer the leverage of a senior engineer's workflow baked into their tooling.

## The Workflow Pattern That Delivers Results

The approach, [detailed by Syncfusion on the .NET Developer Blog](https://devblogs.microsoft.com/dotnet/accelerating-dotnet-maui-with-ai-agents/), introduces a four-phase coordinated agent workflow that handles the complete issue resolution lifecycle:

**Phase 1: Pre-Flight Analysis** - Before any code is changed, the agent reads the GitHub issue, extracts reproduction steps, analyzes the affected components, and identifies platform-specific considerations. This alone eliminates the 30 to 60 minutes a developer typically spends orienting themselves to an unfamiliar issue.

**Phase 2: Test Gate** - The workflow verifies that tests exist for the issue before any fix is attempted. If tests are missing, the workflow stops and directs the developer to create them first using a dedicated test-writing agent. No fix proceeds without a failing test that proves the bug exists.

**Phase 3: Multi-Model Fix Verification** - Multiple independent AI models each propose a distinct fix approach. Each approach is applied and validated against the test suite. Only the approach that passes all tests without introducing regressions is recommended. The key insight here is that having multiple models independently attempt a solution, with each running in complete isolation from the others, surfaces diverse fix strategies and dramatically reduces the chance that a subtle regression slips through.

**Phase 4: Report Generation** - A comprehensive audit-ready summary is produced documenting the issue analysis, each fix approach attempted, test results before and after, and a clear recommendation with rationale.

This is not AI autocomplete. It is a structured engineering methodology embedded in the development workflow.

## The Key Components You Can Use Today

### The pr-review Skill

The [pr-review skill](https://github.com/dotnet/maui/blob/main/.github/skills/pr-review/SKILL.md) implements the four-phase workflow from the GitHub Copilot CLI. A developer invokes it with a natural language prompt:

```
Fix issue #1234
```

or, with additional context for complex scenarios:

```
Fix issue #1234. The problem appears in the data access layer during
concurrent requests. Previous attempts may have missed the async
cancellation token handling.
```

The skill reads the issue, verifies test coverage, runs fix attempts, and produces the report - all autonomously.

### The write-tests-agent

The [write-tests-agent](https://github.com/dotnet/maui/blob/main/.github/agents/write-tests-agent.md) is the critical enabler of the test-first philosophy. It analyzes a GitHub issue, determines the appropriate test strategy - whether UI tests, unit tests, or integration tests - and writes the test code. Critically, it then verifies that the new tests *fail* against the unmodified codebase, proving the tests actually detect the defect rather than passing vacuously.

This addresses one of the most persistent quality problems in government development: tests that were written after the fix and pass regardless of whether the bug is present. A test that does not fail without the fix does not actually test anything.

### The dotnet/skills Marketplace

Microsoft's [dotnet/skills repository](https://github.com/dotnet/skills) provides a growing collection of agent skills built by the .NET platform team. As Tim Heuer [explained on the .NET Developer Blog](https://devblogs.microsoft.com/dotnet/extend-your-coding-agent-with-dotnet-skills/), these are not generic prompts - they are workflows tested and refined by the team that ships the platform itself.

Adding the marketplace takes a single command in the GitHub Copilot CLI:

```
/plugin marketplace add dotnet/skills
```

Skills are supported in GitHub Copilot CLI, VS Code with the Copilot extension, and Visual Studio 2026. The underlying [Agent Skills specification](https://agentskills.io) is an open standard, meaning your team's investment in building custom skills is not locked to a single vendor's tooling.

## Why This Matters for Government

Government software delivery operates under a level of scrutiny that the private sector rarely matches. A production defect in a permitting portal or a benefits calculation system can mean a public complaint, an audit finding, or disrupted services for residents who have no alternative provider. The workflow pattern described here directly addresses several pressure points unique to government:

**Institutional Memory That Doesn't Quit** - When your senior .NET architect retires, the pr-review skill still knows how to analyze a bug methodically, and the write-tests-agent still knows what a good failing test looks like. Agent skills that embed repository-specific context become a form of institutional knowledge that transfers automatically to new team members.

**Built-In Change Management Documentation** - Many government agencies require change management records documenting what was changed, why, and what testing was performed. The report generated by the pr-review workflow provides exactly this - automatically, for every fix. What used to require a developer to write a separate change request memo is now produced as a byproduct of the fix workflow itself.

**Retroactive Test Coverage for Legacy Systems** - Many government agencies are running .NET applications written before modern testing practices were common. The write-tests-agent can be directed at your highest-priority open defects to retroactively add test coverage. This systematically improves the reliability of legacy systems without requiring a full rewrite or a dedicated testing sprint.

**Doing More With Existing Staff** - A 50 to 70 percent reduction in issue resolution time means your existing team can close significantly more issues in the same period - an especially valuable outcome when hiring is constrained. Developers spend less time on investigative work and more time delivering value.

**Quality Access for Junior Developers** - Not every government development team has senior .NET architects available for every pull request review. AI agent skills give junior developers access to expert-level review workflows, narrowing the gap between team experience levels without requiring expensive external training.

## Practical Steps for Getting Started

Adopting this pattern does not require a platform migration or a new procurement action. Here is a practical on-ramp:

**Step 1: Enable GitHub Copilot CLI.** If your organization has GitHub Copilot licenses, enable the CLI by following the [GitHub Copilot in the CLI documentation](https://docs.github.com/en/copilot/github-copilot-in-the-cli). GitHub Enterprise Cloud for Government provides FedRAMP-authorized access to Copilot capabilities for qualifying government organizations.

**Step 2: Add the dotnet/skills marketplace.** From within your .NET project repository:

```
/plugin marketplace add dotnet/skills
/plugin marketplace browse dotnet-agent-skills
```

Browse the available plugins and install the ones relevant to your stack.

**Step 3: Start with test coverage.** Before enabling automated fix workflows, use write-tests-agent to write failing tests for your ten most critical open defects. This alone meaningfully improves your codebase quality and gives the pr-review skill a test suite to work with.

**Step 4: Establish the test-first norm.** Amend your team's contributing guidelines: no fix is merged without a failing test first. Use the pr-review skill's built-in gate to reinforce this norm automatically.

**Step 5: Build custom skills for your patterns.** The Agent Skills specification is designed for extensibility. Your team can create skills tailored to your environment and stored in your repository's `.github/skills/` directory. Examples relevant to government development include:

- A `security-review` skill that checks for common anti-patterns such as hardcoded connection strings, missing Azure Managed Identity configurations, or absent input validation
- A `compliance-check` skill that flags potential violations of your agency's coding standards before a pull request is submitted
- A `dependency-audit` skill that verifies new NuGet packages have acceptable licenses and no known critical security advisories

**Step 6: Connect to Azure Developer CLI for deployment.** Once your code quality discipline is in place, use the [Azure Developer CLI (`azd`)](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview) to standardize provisioning and deployment. The `azd up` command handles infrastructure provisioning, packaging, and deployment in a single step, and integrates cleanly with GitHub Actions for CI/CD pipelines - completing the loop from faster issue resolution to faster delivery.

## Considerations for M365 GCC and Azure Government Environments

Government teams operating on M365 GCC tenants and Azure US Government should verify that any AI services invoked by custom agent skills connect to FedRAMP-authorized endpoints. GitHub Copilot CLI operates against GitHub.com infrastructure, which is covered by FedRAMP Moderate authorization for GitHub Enterprise Cloud for Government customers.

When authoring custom skills that call Azure AI services, configure those calls to use the appropriate `*.usgovcloudapi.net` endpoints to ensure data residency requirements are met. For teams operating in higher security tiers such as GCC High or environments subject to DoD impact level requirements, coordinate with your information system security officer before deploying AI-assisted development workflows, as model interaction policies require agency-specific review.

## Getting Your Team Moving

The 50 to 70 percent reduction in issue resolution time achieved by the .NET MAUI team is not a product demonstration - it is the result of applying consistent engineering discipline to AI tooling. The key insight is that AI agents perform best when given structured context, clear verification criteria, and a defined workflow - not open-ended prompts.

For government development teams, this pattern offers a practical path to faster delivery, stronger code quality, and more resilient institutional knowledge. It works with the .NET applications you already have, the GitHub repositories your team already uses, and the developers already on your payroll.

The tools are available today. The workflow is documented and proven. Starting with a single failing test for your most critical open issue is all it takes to begin.

---

*Resources referenced in this post:*
- [Accelerating .NET MAUI Development with AI Agents - .NET Developer Blog](https://devblogs.microsoft.com/dotnet/accelerating-dotnet-maui-with-ai-agents/)
- [Extend your coding agent with .NET Skills - .NET Developer Blog](https://devblogs.microsoft.com/dotnet/extend-your-coding-agent-with-dotnet-skills/)
- [dotnet/skills repository on GitHub](https://github.com/dotnet/skills)
- [GitHub Copilot in the CLI documentation](https://docs.github.com/en/copilot/github-copilot-in-the-cli)
- [Azure Developer CLI overview - Microsoft Learn](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/overview)
- [Agent Skills specification](https://agentskills.io)
