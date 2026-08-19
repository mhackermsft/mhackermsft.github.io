---
title: 'Managed Instance on App Service: A Managed Home for Your Legacy Windows Web Apps'
date: 2026-08-19T13:52:00+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
categories:
- App Modernization
summary: Managed Instance on Azure App Service gives government IT teams a way to move infrastructure-bound Windows web apps toward a managed platform without rewriting code, while preserving supported COM, registry, MSI, and file-share dependencies where the service is available.
draft: false
image_prompt: Professional illustration of a state or local government IT team modernizing legacy Windows web applications into Azure App Service Managed Instance, with visual cues for COM components, registry settings, Key Vault, Azure Storage, private networking, and secure cloud operations.
image: cover.png
audio: audio.mp3
---

Many government web applications were built a decade or more ago and still run the daily business of a city or state agency. They work, they are trusted, and they are also hard to modernize. They may register COM components, read from the Windows registry, run an MSI during installation, or depend on a mapped drive or legacy file share. Standard Azure App Service is a strong fit for modern, stateless applications, but host-level customization for these legacy dependencies has historically pushed teams back toward virtual machines or full rewrites.

Managed Instance on Azure App Service is designed for this gap. According to the Azure App Service Managed Instance overview on Microsoft Learn, updated August 18, 2026, Managed Instance adds plan-scoped customization for legacy or infrastructure-bound Windows web apps that require Component Object Model (COM), registry access, Microsoft or Windows Installers (MSI), drive mapping, or stricter network boundaries. In practical terms, it gives teams a managed App Service option for a class of Windows workloads that previously looked too infrastructure-bound for platform as a service.

## What Managed Instance actually is

Think of Managed Instance as a middle tier between multitenant App Service and a fully isolated App Service Environment. You still get App Service platform capabilities such as managed identity, virtual network integration, CI/CD integration, Microsoft Entra ID authentication, logging, and platform-managed operations. What you gain is a plan-scoped boundary and controlled operating-system customization hooks for legacy dependencies.

The Microsoft Learn Managed Instance documentation is explicit about the supported surface:

- **Supported:** Windows web workloads, COM registration, registry modifications, MSI installers, storage mounts, UNC paths, scripted drive mapping, managed identity, virtual network integration, MSMQ client and other supported Windows roles or components, CI/CD, and App Service Authentication with Microsoft Entra ID.
- **Not supported:** Linux, containers, App Service Environment, WebJobs, TCP or NetPipes workloads, domain join, NTLM, and Kerberos.
- **Required SKUs:** Pv4 and Pmv4 pricing plans only.
- **Current documented regions:** East Asia, West Central US, North Europe, East US, Australia East, Central India, and South India.

That SKU and region list matters for planning. If you are budgeting or sequencing a migration, model capacity against Pv4 or Pmv4 from the start and confirm regional availability in the target subscription before you schedule production work.

## The migration architecture

The core idea is that host customization should be declared and repeatable, not created by clicking around on a server. Managed Instance supports Remote Desktop Protocol access through Azure Bastion for diagnostics, but Microsoft documentation is clear that persistent changes must be scripted. Changes made during an RDP session are temporary and can be lost after restart, recycle, or platform maintenance.

Three mechanisms carry most migration work.

**1. Configuration install scripts.** Managed Instance uses a zipped configuration script package stored in Azure Storage and accessed by managed identity. The root of that .zip file contains `Install.ps1`, which runs at instance startup. Put installers, DLLs, and supporting configuration files in the same package so the script can reference them through `$PSScriptRoot`.

A representative script looks like this:

```powershell
# Install.ps1 - place at the root of the Managed Instance configuration script .zip
$ErrorActionPreference = 'Stop'

$componentInstaller = Join-Path $PSScriptRoot 'LegacyComponent.msi'
$install = Start-Process -FilePath 'msiexec.exe' -ArgumentList "/i `"$componentInstaller`" /qn /norestart" -Wait -PassThru
if ($install.ExitCode -ne 0) {
    throw "LegacyComponent.msi failed with exit code $($install.ExitCode)."
}

$comDll = Join-Path $PSScriptRoot 'LegacyComServer.dll'
$registration = Start-Process -FilePath 'regsvr32.exe' -ArgumentList "/s `"$comDll`"" -Wait -PassThru
if ($registration.ExitCode -ne 0) {
    throw "LegacyComServer.dll registration failed with exit code $($registration.ExitCode)."
}
```

Keep these scripts idempotent. Check whether a component is already installed before reinstalling it, avoid destructive changes to protected Windows directories, and test each script in a nonproduction Managed Instance plan before using it for a production migration.

**2. Registry adapters.** Some applications expect configuration values in the Windows Registry. Managed Instance supports plan-level registry key adapters, including secret values backed by Azure Key Vault. That keeps connection strings and credentials out of scripts and source control while still satisfying legacy code that reads from the registry.

**3. Storage mounts and UNC access.** Legacy components that expect Azure Files, custom UNC paths, network shares, or scripted drive mappings can use Managed Instance storage mounts. Credentials belong in Key Vault, and network connectivity should be validated through virtual network integration, private endpoints, firewall rules, and DNS before the application cutover.

For diagnostics, use RDP through Azure Bastion to inspect logs, Event Viewer, IIS Manager, and runtime behavior. Do not use RDP as the configuration mechanism. If a diagnostic session reveals a required change, put that change back into `Install.ps1`, a registry adapter, or a storage mount definition.

## A deployment walkthrough

A typical migration follows a predictable path:

1. **Inventory the dependencies.** Catalog COM registrations, registry keys, MSI installers, Windows features, GAC assemblies, scheduled tasks, file-share references, and hard-coded paths. This inventory becomes your configuration script, registry adapter list, and storage mount plan.
2. **Confirm region and SKU availability.** Managed Instance is generally available for Windows web apps in select regions and requires Pv4 or Pmv4. Confirm that the target region exposes the required pricing plan and quota before you create a delivery schedule.
3. **Create the Managed Instance plan and identity.** Assign a managed identity to the App Service plan so platform-level operations can retrieve the configuration script package from Azure Storage and read Key Vault values for registry and storage adapters.
4. **Package the configuration script.** Create a .zip file whose root contains `Install.ps1` and any required installers or supporting files. Grant the plan identity Storage Blob Data Reader access to the script package location.
5. **Externalize secrets to Key Vault.** Store registry adapter values and storage mount credentials in Key Vault. Grant only the required read permissions to the identity used by Managed Instance.
6. **Deploy through CI/CD.** Deploy the app package and the Managed Instance configuration package through a repeatable pipeline. Azure App Service supports GitHub Actions, Azure DevOps, zip deploy, package deploy, and run-from-package patterns.
7. **Validate operations before production.** Validate Managed Instance plan logs, configuration script logs, storage and registry adapter logs, Azure Monitor integration, Application Insights where appropriate, and TLS certificate handling before production adoption. Treat that operational validation as part of the migration, not a post-launch cleanup item.

Adding or editing Managed Instance plan adapters can restart the underlying plan instances and affect all web apps deployed to the plan. Schedule those changes the same way you would schedule other production-impacting platform updates.

## When to choose Managed Instance over the alternatives

Managed Instance is a good fit when the application is a Windows web workload with dependencies such as COM, registry modifications, MSI installers, supported Windows roles or supporting components, GAC assemblies, storage mounts, UNC paths, or drive mapping. The workload still needs to be a web app. Managed Instance is not a way to host non-web service workloads on App Service.

Choose a regular multitenant App Service plan when the app does not need host-level customization, when you need Linux or container support, or when the priority is broad runtime flexibility with the lowest operational overhead.

Choose App Service Environment v3 when you need fully isolated, dedicated infrastructure at larger scale, broader isolation for many applications, or App Service apps deployed into a network-isolated environment. ASE v3 also has documented Azure Government regional availability in US Gov Arizona, US Gov Texas, US Gov Virginia, US DoD Central, and US DoD East, with availability zone support in US Gov Virginia.

Where do containers fit? If your team is ready to containerize and the application is stateless, standard App Service for containers or Azure Container Apps can provide a clean long-term operating model. But containerizing a COM-and-registry-bound application is often its own project. Managed Instance gives teams a path to move first and modernize in phases, rather than making a rewrite the entry ticket to cloud migration.

## Azure Government availability considerations

Azure App Service itself is available in Azure Government, and the Azure Government documentation lists the App Service endpoint suffix as `azurewebsites.us` rather than `azurewebsites.net`. The same comparison guidance also reminds developers that some services and features available in global Azure may not be available in Azure Government, and that sensitive or restricted information should not be placed in Azure resource names.

Managed Instance requires a more specific check. As of the Microsoft Learn Managed Instance documentation updated August 18, 2026, the documented Managed Instance regions are East Asia, West Central US, North Europe, East US, Australia East, Central India, and South India. That list does not include Azure Government regions.

For state and local government teams, the practical guidance is straightforward: validate Managed Instance in the exact cloud and region where the workload must run before committing the migration plan. If a workload must stay in Azure Government and Managed Instance is not available in the required region, evaluate App Service Environment v3, standard Windows App Service where dependencies allow, or a staged migration that keeps the infrastructure-bound component on Azure virtual machines while modernization proceeds.

## Why This Matters for Government

Legacy line-of-business web applications are the connective tissue of local and state government: permitting portals, records systems, internal workflow tools, and case-management applications. They often carry operational risk because they depend on aging servers, specialized installers, undocumented registry settings, or file shares that only a small group of administrators understands.

Managed Instance changes the modernization conversation for eligible workloads. It lets an agency move suitable Windows web apps toward a managed, network-integrated App Service platform while preserving supported infrastructure dependencies that would otherwise block migration. Secrets can move to Key Vault, identity can move to managed identity and Microsoft Entra ID, and instance configuration can become reproducible through scripts and plan-level adapters rather than personal memory.

That matters for compliance-driven modernization. Azure Government documentation notes that Azure and Azure Government are assessed and authorized at FedRAMP High, and that Azure Government adds contractual commitments around US data storage and screened US persons. Even when a specific workload runs in global Azure, the same planning discipline applies: confirm cloud and region availability, avoid sensitive information in resource names, and document how identity, secrets, network paths, and operational controls support the agency's requirements.

For a CIO or IT director, that supports a more defensible modernization roadmap: reduce exposure from aging infrastructure first, then refactor deeper application dependencies over time. For engineering teams, it creates a paved path for applications that are valuable, trusted, and not yet ready for a rewrite.

Start with one noncritical Windows web app. Build the dependency inventory, package `Install.ps1`, externalize secrets to Key Vault, validate storage and registry adapters, and confirm regional availability before expanding the pattern across the portfolio.
