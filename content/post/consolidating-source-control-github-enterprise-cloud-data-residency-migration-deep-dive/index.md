---
title: 'Consolidating Source Control on GitHub Enterprise Cloud With Data Residency: A Migration Deep Dive'
date: 2026-09-02T17:45:07+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
- Announcements
categories:
- App Modernization
summary: How government IT teams can move repositories, pull requests, work item links, and history from Azure DevOps Cloud and GitHub Enterprise Server onto GitHub Enterprise Cloud with data residency, using GitHub Enterprise Importer and Enterprise Live Migrations.
draft: false
image_prompt: A secure government software modernization scene showing repositories flowing from Azure DevOps and an on-premises GitHub Enterprise Server appliance into GitHub Enterprise Cloud with data residency in the US region, with identity, audit, and network controls represented as clean architectural icons.
image: cover.png
audio: audio.mp3
---

Many government engineering organizations have accumulated source control sprawl over the last decade: some teams on Azure DevOps, others on a self-hosted GitHub Enterprise Server (GHES) appliance, and a few still on legacy version control. Consolidating onto a single, managed developer platform can reduce operational overhead, unify identity and audit, and give developers access to current capabilities without waiting for on-premises upgrade cycles. The public sector question is always the same: where does the data live, and can we prove it?

GitHub Enterprise Cloud with data residency addresses that question directly. For migration planning, the key 2026 change is Enterprise Live Migrations reaching general availability for the GHES-to-GHE.com path. GitHub Enterprise Importer remains the primary Azure DevOps Cloud migration path. This post walks through the two migration engines you are most likely to use, a hands-on Azure DevOps Cloud walkthrough, the residency architecture, and what government teams should validate before committing.

## Two migration paths, two different tools

There is a common misconception that a single tool moves every source system in the same way. In practice, GitHub provides complementary migration engines, and choosing correctly per repository is the difference between a smooth cutover and a painful one.

**GitHub Enterprise Importer (GEI)** is the customizable, repository-by-repository tool for migrating into GitHub Enterprise Cloud from supported sources. Current GitHub documentation lists support for Azure DevOps Cloud, Bitbucket Server and Bitbucket Data Center 5.14+, GitHub.com, GitHub Enterprise Server 3.4.1+, and GitLab. For Azure DevOps, GEI supports Azure DevOps Cloud, not Azure DevOps Server. It can migrate Git source including commit history, pull requests, user history for pull requests, work item links on pull requests, attachments on pull requests, and repository branch policies subject to documented limitations. It also supports custom trial runs as many times as needed before production, and it uses clear, unblocking error logging so a non-critical issue such as one pull request comment failing to migrate does not stop the whole job.

**Enterprise Live Migrations (ELM)** is the live migration engine for the GHES-to-GHE.com path. As announced on the GitHub Changelog on September 1, 2026, ELM reached general availability for migrations from GitHub Enterprise Server to GitHub Enterprise Cloud with data residency (GHEC DR). ELM continuously syncs supported repository data from the source appliance to the target so developers can keep working during most of the migration, and the final cutover requires the time needed to drain remaining in-flight changes. Current ELM documentation positions it for large monorepos, deep Git history, high volumes of issues and pull requests, and repositories with activity around the clock.

The important nuance: ELM is a GHES-to-GHEC DR path, not an Azure DevOps path. If your consolidation program includes both Azure DevOps Cloud and a GHES appliance, use GEI for the Azure DevOps side and ELM for the GHES side. GitHub positions the two as complementary, and the September 2026 changelog says they can run concurrently as part of one migration strategy. Use GEI where a brief downtime window is acceptable, and use ELM where the repository needs the lowest-disruption approach. For ELM, plan around the documented concurrency limits: up to 10 concurrent repository migrations from a single GHES instance and 20 concurrent migrations per destination enterprise.

## Where the data lives: the residency model

With GitHub Enterprise Cloud with data residency, your enterprise is hosted on a dedicated subdomain of GHE.com, for example `octocorp.ghe.com`, and you choose the region where your company code and data are stored. Current GitHub documentation lists the available data residency regions as EU, Australia, US, and Japan, with more planned.

For US state and local agencies, the US region is an important control because documented categories of customer content and identifying data are stored within the chosen region. That includes repositories, repository names, source code, pull requests, comments, file paths, raw URLs, filenames, and certain data or logs that identify a company or person. It is not the same thing as saying every operational or commercial data element remains in-region. GitHub documentation states that some data may be stored or transferred outside the chosen region, including certain billing, purchase, payment, support, feedback, telemetry, GitHub Copilot, and secret scanning data depending on configuration and feature use. Treat the residency region as a strong data location control, not as a substitute for your own regulatory review.

A few architectural characteristics matter for government reviewers:

- **Enterprise Managed Users (EMU).** Enterprises on GHE.com use managed accounts provisioned and authenticated through an identity provider. GitHub supports SCIM for provisioning and SAML or OIDC for authentication with supported partner IdPs. Managed user accounts can only access the enterprise resources they are authorized to access and cannot create public content or collaborate outside the enterprise.
- **Dedicated API surface.** REST and GraphQL integrations must send requests to the enterprise dedicated URL on GHE.com. For example, if the subdomain is `octocorp`, API requests use `https://api.octocorp.ghe.com` rather than the shared GitHub.com API endpoint.
- **Distinct network details.** Hostnames, IP ranges, SSH usage, and SSH key fingerprints for GHE.com differ from GitHub.com. Client systems, migration hosts, identity provider integrations, and storage access paths should be allowlisted based on the GHE.com network details, not copied from an existing GitHub.com allowlist.

## Hands-on: migrating Azure DevOps Cloud repositories with GEI

GEI is driven by a GitHub CLI extension. For Azure DevOps, the extension is `ado2gh`, which can generate a PowerShell migration script covering repositories in a source organization.

Start by generating the script:

```powershell
gh ado2gh generate-script `
  --ado-org SOURCE `
  --github-org DESTINATION `
  --output migrate.ps1 `
  --all `
  --download-migration-logs
```

Replace `SOURCE` with the name of the Azure DevOps source organization and `DESTINATION` with the target GitHub organization. The `--all` flag adds functionality to the generated script, such as rewiring pipelines, creating teams, and configuring Azure Boards integrations. The `--download-migration-logs` flag downloads a migration log for each migrated repository so the team has an audit trail for review.

The critical flag for a data residency target is `--target-api-url`. When your destination is a GHE.com subdomain, point the tooling at the enterprise API base URL:

```powershell
gh ado2gh generate-script `
  --ado-org SOURCE `
  --github-org DESTINATION `
  --output migrate.ps1 `
  --target-api-url https://api.octocorp.ghe.com
```

Without the GHE.com API base URL, your migration script will not be targeting the residency-scoped enterprise you intended. GitHub documentation shows this URL pattern for GHE.com API access.

Review the generated script before touching production. It contains one command per repository, so you can remove or comment out repositories you do not want to move, rename a repository at the destination with `--target-repo`, or override visibility with `--target-repo-visibility`. By default, the generated script sets the same visibility as the source repository.

GitHub strongly recommends a trial run. Create a test organization, using a suffix such as `-sandbox` for clarity, run the migration against it, and let repository owners validate the result on their own schedule. Trial runs can happen at any time and do not require teams to halt work.

When you are ready for production, execute the reviewed script in PowerShell:

```powershell
.\migrate.ps1
```

Plan the production window carefully. GEI does not support delta migrations, so changes made during or after the migration have to be migrated manually. GitHub recommends halting work in the repositories being migrated. That limitation is exactly why ELM matters for the GHES path: repositories that cannot tolerate a pause should be evaluated for live migration instead of a conventional GEI cutover.

## Migration architecture patterns for a consolidation program

For a multi-source consolidation, a phased pattern keeps risk contained:

1. **Inventory and classify.** Score each repository by size, history depth, issue and pull request volume, Git LFS usage, branch policy complexity, and activity level. Low-activity Azure DevOps Cloud repositories are GEI candidates with a short downtime window. High-activity monorepos on GHES are ELM candidates.
2. **Stand up the GHEC DR enterprise.** Choose the US region if domestic storage is part of the control objective, configure Enterprise Managed Users with your identity provider, and update network allowlists for GHE.com hostnames, IP ranges, SSH patterns, and API endpoints.
3. **Prove it in a sandbox.** Use GEI trial runs for Azure DevOps Cloud migrations. For GHES repositories using ELM, use repository-level progress tracking so operators can see failures before cutover and make a go or no-go decision with evidence.
4. **Cut over by wave.** For GEI waves, schedule a maintenance window and halt work. For ELM waves, let continuous sync run and cut over when the remaining in-flight changes have drained.
5. **Complete follow-up tasks.** For GEI, review migration logs, set repository visibility, reclaim mannequins, push Git LFS objects when required, and configure Azure Pipelines or Azure Boards integrations. For ELM, restore user access, reclaim mannequins, reattribute Git activity by aligning commit email addresses through the identity provider, and recreate organization-level settings such as teams, projects, and webhooks.

Manage ELM through the `gh elm` CLI extension, which the September 2026 GitHub Changelog describes as the interface for configuring credentials and managing the migration lifecycle through the GHES REST API. ELM migrates almost all repository-level data, but organization-level resources are excluded and must be configured manually on the target. Repository rulesets are not migrated, and branch protections are only partially migrated, so review branch protection rules before allowing users to work in the destination repository.

Do not rely on a generic GHES version statement. Current ELM preparation documentation says the instructions assume supported patch releases for GHES 3.17 and later, and the patch floor differs by release line. Confirm the exact supported patch level in the ELM preparation docs and your GHES release notes before beginning.

## Why This Matters for Government

Source control is where an agency's intellectual property, automation logic, and infrastructure-as-code increasingly live. Consolidating onto GitHub Enterprise Cloud with data residency gives government IT leaders three controls they consistently need for modernization programs.

First, **data location control**: selecting the US region keeps documented categories of customer content and user data in the chosen region, which can support evidence collection for data residency reviews. Second, **identity isolation and auditability**: Enterprise Managed Users let the agency manage account lifecycle and authentication from its identity provider, limit managed users to enterprise resources, and centralize governance and audit activity. Third, **reduced operational burden**: retiring a self-hosted GHES appliance removes patch, upgrade, and maintenance windows from the agency workload while giving developers access to current GitHub Enterprise Cloud features.

For state and local government, the compliance question should be framed carefully. Data residency can support conversations about StateRAMP, FedRAMP-related system boundaries, CJIS, IRS Publication 1075, HIPAA, and agency-specific records policies, but it does not automatically satisfy those obligations. Before migrating regulated repositories, confirm with your compliance office, security team, legal counsel, and procurement stakeholders that the chosen residency region, identity model, logging posture, network access model, and platform authorization posture meet the specific requirements for the workloads involved.

The migration tooling now makes consolidation practical rather than aspirational. GEI gives teams repeatable Azure DevOps Cloud trial runs before production, and ELM general availability means the largest, always-on GHES repositories can move to GHEC DR with a cutover measured in minutes when the source environment is on a supported release.

## Get started

- [About GitHub Enterprise Importer](https://docs.github.com/en/migrations/using-github-enterprise-importer/understanding-github-enterprise-importer/about-github-enterprise-importer)
- [Understand migrations from Azure DevOps to GitHub](https://docs.github.com/en/migrations/using-github-enterprise-importer/migrating-from-azure-devops-to-github-enterprise-cloud/about-migrations-from-azure-devops-to-github-enterprise-cloud)
- [Migrate your repositories from Azure DevOps to GitHub](https://docs.github.com/en/migrations/using-github-enterprise-importer/migrating-from-azure-devops-to-github-enterprise-cloud/migrating-repositories-from-azure-devops-to-github-enterprise-cloud)
- [Azure DevOps migration follow-up tasks](https://docs.github.com/en/migrations/ado/follow-up-tasks)
- [About GitHub Enterprise Cloud with data residency](https://docs.github.com/en/enterprise-cloud@latest/admin/data-residency/about-github-enterprise-cloud-with-data-residency)
- [About storage of your data with data residency](https://docs.github.com/en/enterprise-cloud@latest/admin/data-residency/about-storage-of-your-data-with-data-residency)
- [Network details for GHE.com](https://docs.github.com/en/enterprise-cloud@latest/admin/data-residency/network-details-for-ghecom)
- [Feature overview for GitHub Enterprise Cloud with data residency](https://docs.github.com/en/enterprise-cloud@latest/admin/data-residency/feature-overview-for-github-enterprise-cloud-with-data-residency)
- [About Enterprise Managed Users](https://docs.github.com/en/enterprise-cloud@latest/admin/identity-and-access-management/using-enterprise-managed-users-and-saml-for-iam/about-enterprise-managed-users)
- [About live migrations from GitHub Enterprise Server to GHE.com](https://docs.github.com/en/migrations/elm/about-live-migrations)
- [Enterprise Live Migrations CLI reference](https://docs.github.com/en/migrations/elm/elm-cli-reference)
- [Migrated data for live migrations](https://docs.github.com/en/migrations/elm/migrated-data-reference)
- [Enterprise Live Migrations from GHES to ghe.com generally available](https://github.blog/changelog/2026-09-01-enterprise-live-migrations-from-ghes-to-ghe-com-generally-available/)
