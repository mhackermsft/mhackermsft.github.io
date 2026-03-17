---
title: 'AI Agents as Your Modernization Team: Accelerating Legacy Application Migration in Government'
date: 2026-03-13T12:32:54+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- Announcements
- How To
categories:
- App Modernization
summary: Microsoft's agentic end-to-end modernization toolchain - combining Azure Migrate AppCAT, GitHub Copilot App Modernization, and AI-driven wave planning - gives government IT teams an intelligent assistant that can assess, plan, and execute legacy application migrations at a scale and speed that manual efforts simply cannot match.
draft: false
image_prompt: A government IT operations center with multiple screens showing Azure cloud dashboards, code assessment reports, and wave migration planning visualizations. Subtle blue and white Microsoft Azure color palette. Professional, clean, modern aesthetic suitable for a government technology audience.
image: cover.jpg
---

The legacy application problem in government is not a secret. Decades of layered solutions - ASP.NET WebForms portals from the mid-2000s, Java EE monoliths running on application servers well past their vendor support lifecycle, custom .NET Framework 3.5 services that nobody fully understands anymore - sit at the foundation of mission-critical workflows. Every year, these systems accumulate more technical debt, more unpatched vulnerabilities, and more distance from the modern platforms that support Zero Trust security, cloud scalability, and developer agility.

The hard truth is that traditional modernization approaches - manual code audits, months of architecture planning, lengthy rewrites - are not keeping pace with the scope of the problem. A government IT department with a portfolio of 150 applications cannot realistically dedicate a team to manually assess, refactor, and re-test every one of them. That is where AI-powered agentic modernization changes the equation.

Microsoft has been assembling a cohesive end-to-end toolchain that puts AI agents to work at every stage of the modernization lifecycle: assessment, planning, and execution. For government IT leaders, this represents a genuine inflection point.

## The Three-Phase Agentic Modernization Model

### Phase 1 - Assessment: Let AI Read the Code First

Before any meaningful modernization plan can be built, someone - or something - needs to understand what existing applications actually do, what dependencies they carry, and what cloud readiness challenges they face. Historically, that assessment work fell entirely on developers and architects, requiring weeks of manual code reviews.

Microsoft's [Azure Migrate Application and Code Assessment (AppCAT)](https://learn.microsoft.com/en-us/azure/migrate/appcat/overview) automates this discovery across both .NET and Java application portfolios, performing deep static analysis of source code, binaries, and configuration files to surface:

- **Technology inventory**: What frameworks, libraries, and runtimes does the application use? This is especially valuable for legacy applications with minimal documentation - which describes the majority of long-running government systems.
- **Cloud readiness issues**: What will break, degrade, or need re-architecting when the application moves to Azure App Service, Azure Kubernetes Service (AKS), or Azure Container Apps?
- **Effort estimates**: Which issues are high-impact versus low-impact, and how complex is the remediation work?

For .NET applications, [AppCAT for .NET](https://learn.microsoft.com/en-us/azure/migrate/appcat/dotnet) integrates directly into Visual Studio and the .NET CLI. Developers can right-click a project or solution and select **Re-platform to Azure** to trigger a full analysis without leaving their IDE. For Java shops, [AppCAT for Java](https://learn.microsoft.com/en-us/azure/migrate/appcat/java) - which reached general availability in July 2025 - supports Java EE, Spring Boot, and Jakarta EE applications with configurable rulesets tailored to specific Azure target platforms.

Critically, AppCAT performs analysis locally. Source code does not need to leave your environment, which aligns with the data handling constraints that govern code security in government settings.

### Phase 2 - Planning: Wave Planning for Portfolio-Scale Migrations

One of the most underappreciated challenges in large-scale government modernization is sequencing. Modernizing a single application in isolation is straightforward. Modernizing 80 applications while keeping dependent services running, respecting procurement timelines, and aligning with budget cycles is a fundamentally different problem.

[Azure Migrate Wave Planning](https://learn.microsoft.com/en-us/azure/migrate/overview) provides a structured framework for managing exactly this complexity. Wave Planning allows migration teams to:

- **Group workloads into logical waves** based on business criticality, technical complexity, and inter-application dependencies identified through dependency analysis
- **Visualize execution timelines** across all waves so leadership and program managers can see the full migration roadmap at a glance
- **Track status in real time** through preparation, testing, and completion stages for each workload
- **Integrate external migration tools** - Wave Planning accepts data from Azure Database Migration Service (DMS), third-party tools, and custom scripts through its extensible tracking capability

For a government organization managing application infrastructure across multiple departments or agencies, the wave-based approach mirrors how capital improvement programs are typically funded and approved - in defined phases, with measurable deliverables at each milestone. It gives program managers the governance structure they need while giving IT teams the technical flexibility they require.

### Phase 3 - Execution: GitHub Copilot as Your AI Modernization Engineer

Assessment tells you what needs to change. Planning tells you in what order. Execution is where AI agents deliver the most dramatic acceleration.

[GitHub Copilot App Modernization for Java](https://learn.microsoft.com/en-us/azure/developer/java/migration/migrate-github-copilot-app-modernization-for-java-quickstart-assess-migrate) is a VS Code and IntelliJ IDEA extension that pairs AppCAT's assessment engine with GitHub Copilot's AI reasoning to automate code remediation. The workflow operates in five stages:

1. **Run the assessment**: The Copilot agent invokes AppCAT internally and generates a categorized report of cloud readiness issues, each paired with recommended solutions and step-by-step remediation guidance.
2. **Select a migration task**: For example, migrating an Azure SQL database connection from username and password authentication to Azure Managed Identity - a Zero Trust-aligned change that previously required a developer to manually locate every connection string, update driver configuration, and validate each code path.
3. **Let the agent execute**: Copilot creates a dedicated migration branch in Git, generates `plan.md` and `progress.md` tracking files for transparency, and begins making targeted code changes.
4. **Iterate through automated validation**: After code changes are applied, the agent runs a multi-step validation loop that builds the project, scans for Common Vulnerabilities and Exposures (CVEs) in updated dependencies, validates functional consistency, and reruns unit tests - automatically fixing issues it encounters.
5. **Review and merge**: Developers review the final diff and accept or modify the changes before merging, maintaining full human oversight over everything that reaches production.

For .NET applications, [GitHub Copilot App Modernization for .NET](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization-overview) provides equivalent capabilities across Visual Studio, Visual Studio Code, and the GitHub Copilot CLI.

The [Java upgrade quickstart](https://learn.microsoft.com/en-us/java/upgrade/quickstart-upgrade) further illustrates the breadth of what the agent handles in a single session: upgrading JDK versions from Java 8 to Java 21 or Java 25, migrating from Spring Boot 2.x to 3.x, converting Java EE to Jakarta EE 10, migrating build systems from Ant to Maven, and automatically remediating CVEs in upgraded dependencies. These are exactly the kinds of pervasive, cross-cutting changes that are tedious and error-prone when done by hand - and that AI agents can apply systematically across an entire codebase.

## Why This Matters for Government

Government agencies face a unique convergence of pressures that makes agentic modernization not just useful, but necessary.

**Security compliance drives urgency.** Legacy applications running on end-of-life runtimes - Java 8, .NET Framework 4.x, outdated Spring Boot releases - accumulate unpatched CVEs that generate audit findings under FISMA, FedRAMP, and equivalent state-level security frameworks. The automated CVE scanning and dependency remediation built into the GitHub Copilot App Modernization workflow directly addresses this exposure - at scale and without requiring manual analysis of hundreds of dependency manifests.

**Zero Trust mandates require modern authentication patterns.** Moving from embedded credentials and shared service accounts to Azure Managed Identity is a core requirement for Zero Trust-compliant architectures. This is precisely the kind of systematic, repetitive code change that AI agents are best suited for - applying the same transformation consistently across every data access point in an application, without the fatigue-related inconsistencies that affect manual refactoring.

**Workforce constraints are real.** Government IT departments compete with the private sector for experienced developers. AI modernization agents effectively multiply the output of the developers you already have, enabling a small team to assess and remediate applications at a pace that would otherwise require a substantially larger staff - or expensive contractor engagements.

**Budget accountability requires documented progress.** Azure Migrate Wave Planning's milestone tracking produces the reporting artifacts that oversight bodies, budget committees, and legislative auditors expect. Every migration wave has a defined start, a defined completion target, and a tracked status - which maps cleanly onto how government capital improvement projects are reported and governed.

**Application portfolios are large and undocumented.** Many government applications were built by contractors who are no longer engaged, using frameworks that predate modern documentation practices. AppCAT's technology discovery capability can reveal what a legacy application actually uses - which is sometimes genuinely unknown to the current IT staff maintaining it.

## Availability and Government Cloud Considerations

For organizations using **Azure commercial** - including agencies on M365 GCC tenants accessing Azure commercial subscriptions - all tools described in this post are generally available today. Azure Migrate, AppCAT for .NET and Java, and the GitHub Copilot App Modernization extensions work with standard Azure subscriptions and local developer tooling.

For organizations running workloads in **Azure US Government** regions, Azure Migrate and Azure Migrate Wave Planning are available in Azure Government. AppCAT for .NET and Java run entirely as local CLI tools or IDE extensions - they do not require cloud connectivity during analysis and are not subject to cloud region restrictions, making them suitable for use in air-gapped or restricted environments.

GitHub Copilot is available to government agencies through standard GitHub Enterprise licensing. Agencies should review their security authorization posture with their information system security officers (ISSOs) before routing code through any cloud-connected AI service, particularly for applications classified at higher sensitivity levels.

## Getting Started: A Practical Path Forward

For government IT leaders considering where to begin, a practical sequence looks like this:

1. **Inventory your portfolio with Azure Migrate**: Deploy an Azure Migrate appliance to discover and inventory servers, databases, and web applications running in your environment. Dependency analysis will reveal the application groupings you need for wave planning.

2. **Run AppCAT on your highest-risk applications first**: Prioritize applications with the oldest runtimes, the most significant CVE exposure, or the nearest compliance deadlines. AppCAT reports are generated as HTML, JSON, or CSV and can be directly incorporated into security documentation and POA&M artifacts.

3. **Build a wave plan aligned to your budget cycle**: Use Azure Migrate Wave Planning to group your application portfolio into phased migration waves. Aligning wave boundaries to your annual budget and procurement timelines makes the program more governable and easier to fund incrementally.

4. **Pilot GitHub Copilot App Modernization on a representative application**: Choose a Java or .NET application that is important enough to be representative but low enough risk to tolerate iteration. Walk through the full agentic migration workflow - assessment, task selection, code remediation, validation loop, review - and measure the time investment against your manual modernization baseline.

5. **Establish a code review policy for AI-generated changes**: The agent creates Git branches and tracks all changes in version control by design. Apply your standard pull request review process to AI-generated changes, and ensure that a qualified developer approves every migration before it reaches production.

## The Modernization Opportunity Is Now

The combination of AppCAT's deep code analysis, Azure Migrate's portfolio-scale wave planning, and GitHub Copilot's agentic code remediation represents the most complete AI-assisted modernization toolkit Microsoft has assembled to date. For government organizations carrying years of accumulated technical debt, this is not an incremental improvement - it is a fundamental change in what is achievable with the staff and budget you already have.

Legacy systems do not have to be a permanent liability. With AI agents handling the assessment and remediation work that previously consumed months of skilled developer time, the path from aging on-premises applications to secure, cloud-native deployments on Azure is shorter than it has ever been.

The tools are available today. The question is whether your modernization roadmap is positioned to take advantage of them.

---

**Resources:**
- [Azure Migrate Application and Code Assessment Overview](https://learn.microsoft.com/en-us/azure/migrate/appcat/overview)
- [AppCAT for .NET - Get Started](https://learn.microsoft.com/en-us/azure/migrate/appcat/dotnet)
- [AppCAT for Java - Get Started](https://learn.microsoft.com/en-us/azure/migrate/appcat/java)
- [GitHub Copilot App Modernization for Java - Quickstart](https://learn.microsoft.com/en-us/azure/developer/java/migration/migrate-github-copilot-app-modernization-for-java-quickstart-assess-migrate)
- [GitHub Copilot App Modernization for .NET - Overview](https://learn.microsoft.com/en-us/dotnet/core/porting/github-copilot-app-modernization-overview)
- [Upgrade Java with GitHub Copilot Modernization](https://learn.microsoft.com/en-us/java/upgrade/quickstart-upgrade)
- [Azure Migrate Wave Planning](https://learn.microsoft.com/en-us/azure/migrate/overview)
