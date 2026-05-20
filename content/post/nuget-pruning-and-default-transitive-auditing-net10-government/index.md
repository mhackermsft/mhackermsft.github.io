---
title: 'NuGet Package Pruning and Default Transitive Auditing in .NET 10: A Practical Guide for Government .NET Teams'
date: 2026-05-20T16:20:51+00:00
author: Mike Hacker
tags:
- App Modernization
- Security
- How To
categories:
- App Modernization
summary: How .NET 10's automatic package pruning and default transitive vulnerability auditing reshape supply chain hygiene for public sector .NET shops, with concrete MSBuild and CI/CD examples.
draft: false
image_prompt: A clean, modern illustration showing a layered NuGet package dependency tree being pruned, with a shield icon representing vulnerability scanning, in blue and purple Azure brand colors, suitable for a government technology blog post.
image: cover.png
audio: audio.mp3
---

Public sector .NET teams have spent the last several years fighting an asymmetric battle against transitive NuGet vulnerabilities. You add one direct `PackageReference`, NuGet quietly pulls in fifty more, and a CVE published against a library you've never heard of suddenly shows up in your SBOM the morning your ATO auditor calls.

.NET 10 (LTS, [released November 11, 2025](https://devblogs.microsoft.com/dotnet/announcing-dotnet-10/)) ships with two NuGet 7.0 changes that materially improve this posture for any project that retargets to `net10.0`:

1. **Automatic package pruning** removes transitively referenced packages that are already part of the shared framework, shrinking the dependency graph at restore time.
2. **Default transitive auditing** (`NuGetAuditMode=all`) makes `dotnet restore` raise vulnerability warnings on the entire closure of your dependencies, not just the ones you typed into your `.csproj`.

Both are documented in the official [NuGet 7.0 release notes](https://learn.microsoft.com/en-us/nuget/release-notes/nuget-7.0), which lists "Package pruning is enabled for all projects targeting .NET 10" and "Projects that target .NET 10 warn for vulnerabilities in transitive packages by defaulting to NuGetAuditMode=all" as headline behaviors. NuGet 7.0.3 is the current servicing release as of the .NET SDK 10.0.106 band.

This post is for hands-on developers and tech leads, not executives. We're going to look at exactly what changes in your project file, how to wire pruning into CI, and how to read the resulting warnings.

## What package pruning actually does

The .NET shared framework already ships hundreds of assemblies: `System.Text.Json`, `System.Memory`, `Microsoft.Extensions.*`, `System.Security.Cryptography.*`, and so on. Before .NET 10, if a transitive dependency declared a `PackageReference` to, say, `System.Text.Json 6.0.0`, NuGet faithfully resolved that package and copied it into your output even though `net10.0` already provides a newer in-box implementation.

That extra package was three bad things at once:

- A larger output footprint.
- A second copy of the same surface area, which the runtime had to unify at load.
- An attack surface entry in your SBOM and audit report, often with its own CVE backlog.

With pruning enabled, NuGet looks at the transitive graph and removes references that are subsumed by the target framework. Per the NuGet 7.0 notes, "Pruning privatizes a direct reference by applying `PrivateAssets=all` and `IncludeAssets=none`" (issue #14196), which keeps the dependency declaration honest without flowing it down to consumers of your library.

For `net10.0` projects, pruning is on by default. You don't have to opt in. The first restore against a project that has a `packages.lock.json` will produce a one-time diff to that lock file (also called out as a known behavior in the release notes); commit it and move on.

### Verifying pruning in your own project

Retarget a sample library to `net10.0`:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Some.Third.Party.Lib" Version="4.2.0" />
  </ItemGroup>
</Project>
```

Then run:

```powershell
dotnet restore
dotnet nuget why .\MyLib.csproj System.Text.Json
```

The `dotnet nuget why` command prints the resolution path for a transitive package. On a `net10.0` project with pruning active, you'll see packages that previously appeared in the graph drop out entirely.

### Suppressing or extending pruning manually

If you need to add a package to the prune list yourself, for example because you're targeting an older TFM in a multi-target build, use `PrunePackageReference`:

```xml
<ItemGroup Condition="'$(TargetFramework)' == 'net8.0'">
  <PrunePackageReference Include="System.Text.Json" Version="8.0.0" />
</ItemGroup>
```

For multi-targeted projects, the .NET 10 SDK pruning logic only applies automatically to the `net10.0` TFM. Older TFMs can still benefit, but you opt in explicitly. The NuGet team moved the pruning-enabled framework list into `NuGet.targets` (issue #14424) so it can evolve with future SDKs.

## Default transitive auditing

The second change is just as consequential. Per the [official NuGet auditing documentation](https://learn.microsoft.com/en-us/nuget/concepts/auditing-packages):

> `NuGetAuditMode` defaults to `all` when a project targets `net10.0` or higher. Otherwise `NuGetAuditMode` defaults to `direct`.

In plain English: before .NET 10, `dotnet restore` only warned you about CVEs in packages you directly referenced. Vulnerabilities buried three layers down in the transitive graph were silent unless you flipped a switch. Starting with `net10.0`, every restore evaluates the entire closure against the [GitHub Advisory Database](https://github.com/advisories) and raises the standard audit warnings:

| Warning | Severity |
| --- | --- |
| NU1901 | Low |
| NU1902 | Moderate |
| NU1903 | High |
| NU1904 | Critical |
| NU1905 | Audit source missing vulnerability data |

### Reading and acting on the report

For a typical web API that pulled in a vulnerable transitive package, a `dotnet restore` will now print something like:

```
warning NU1903: Package 'Some.Transitive.Lib' 1.2.3 has a known high severity vulnerability,
  https://github.com/advisories/GHSA-xxxx-yyyy-zzzz
```

Follow the NuGet team's published remediation order:

1. Update the top-level package that pulled it in.
2. If no upstream fix exists, update the closest intermediate package.
3. Add a direct `PackageReference` to a fixed version as a last resort.
4. Or, if you've reviewed and accepted the risk, suppress the specific advisory.

.NET 10 also adds a new convenience command:

```powershell
dotnet package update --vulnerable
```

This upgrades vulnerable packages to the lowest version higher than the currently referenced version that has no known vulnerabilities. It's documented in the [`dotnet package update` reference](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-update) and was added in NuGet 7.0 (issue #13372). The command supports `--interactive`, `--project`, and `--verbosity`; run it against a feature branch and review the resulting diff before merging.

### Suppression done right

When you genuinely cannot patch (an advisory is non-exploitable in your code path, or no fixed version exists yet), suppress the specific advisory rather than disabling auditing globally:

```xml
<ItemGroup>
  <NuGetAuditSuppress Include="https://github.com/advisories/GHSA-xxxx-yyyy-zzzz" />
</ItemGroup>
```

Keep these in a `Directory.Build.props` so they're reviewable across the repository.

## Configuring audit sources for restricted networks

Many government tenants block direct egress to `api.nuget.org`. NuGet supports a vulnerability-only endpoint that network teams are often willing to allow because it doesn't serve package binaries:

```xml
<configuration>
  <auditSources>
    <clear />
    <add key="nuget.org" value="https://data.nuget.org/v3/index.json" />
  </auditSources>
</configuration>
```

Use this when your build agents pull packages from a private upstream (Azure Artifacts, JFrog, Sonatype) but still need authoritative CVE data. This is documented under "Audit Sources" in the official auditing reference.

## Wiring it into your CI/CD pipeline

Once pruning and default auditing are active, the next step is making the build fail loudly on critical findings. Add this to a `Directory.Build.props` at the repo root:

```xml
<Project>
  <PropertyGroup>
    <NuGetAudit>true</NuGetAudit>
    <NuGetAuditMode>all</NuGetAuditMode>
    <NuGetAuditLevel>moderate</NuGetAuditLevel>
    <WarningsAsErrors>$(WarningsAsErrors);NU1903;NU1904</WarningsAsErrors>
  </PropertyGroup>
</Project>
```

This keeps low and moderate findings as informational warnings while turning High and Critical CVEs into hard build failures. Setting `NuGetAuditMode=all` explicitly is belt-and-suspenders for repositories that still contain pre-`net10.0` projects.

A minimal Azure DevOps step:

```yaml
- task: UseDotNet@2
  inputs:
    packageType: 'sdk'
    version: '10.0.x'

- script: dotnet restore --locked-mode
  displayName: 'Restore with audit'

- script: dotnet build --no-restore -c Release
  displayName: 'Build'

- script: dotnet list package --vulnerable --include-transitive --format json > audit.json
  displayName: 'Export vulnerability report'

- task: PublishBuildArtifacts@1
  inputs:
    pathToPublish: 'audit.json'
    artifactName: 'nuget-audit'
```

The `--locked-mode` flag plus a committed `packages.lock.json` gives you reproducible restores, which matters for FedRAMP and StateRAMP evidence collection. Note that pruning interacts with locked mode; the NuGet team improved the NU1004 diagnostic specifically to handle this case (issue #14075).

For GitHub Actions running on GitHub-hosted runners, the equivalent shape uses `actions/setup-dotnet@v4` with `dotnet-version: '10.0.x'` and the same `dotnet` commands.

## Why this matters for government

State and local agencies typically inherit three pressures simultaneously: a CISA known exploited vulnerabilities (KEV) catalog that updates weekly, an internal application security team that wants SBOM evidence on every release, and a procurement timeline that won't tolerate emergency package upgrades the week before a go-live.

Default transitive auditing meaningfully changes the math on the first two. Vulnerabilities surface at the developer's desk during the inner loop, not weeks later in a Defender for Cloud scan or a third-party SAST report. That shortens the mean time to remediate and shrinks the window where a publicly disclosed CVE sits unpatched in production.

Pruning helps with the third pressure. A smaller, cleaner dependency graph means fewer line items in the SBOM you hand to auditors, fewer false positives from scanners that flag the redundant in-box package copies, and a smaller deployable artifact for environments where bandwidth to gov cloud regions is constrained.

Both Azure commercial and Azure US Government already support .NET 10 across App Service, Azure Functions, Azure Container Apps, and AKS. The NuGet tooling described here lives inside the SDK and runs on the build agent regardless of where the workload eventually deploys, so behavior is identical in commercial and US Government cloud.

## Practical adoption checklist

For a typical agency .NET portfolio, a sensible rollout looks like:

1. Install the .NET 10 SDK on developer machines and build agents. .NET 10 is LTS, supported through November 10, 2028.
2. Retarget one non-critical service to `net10.0` and commit the resulting `packages.lock.json` diff.
3. Add the `Directory.Build.props` snippet above to elevate High and Critical warnings to errors.
4. Run `dotnet list package --vulnerable --include-transitive` against the rest of the portfolio to inventory exposure before you migrate the next workload.
5. For shared internal libraries, audit any `PrunePackageReference` declarations and ensure your library doesn't ship a hidden dependency on a package it expected callers to provide.

## References

- [Announcing .NET 10 (November 2025)](https://devblogs.microsoft.com/dotnet/announcing-dotnet-10/)
- [NuGet 7.0 Release Notes](https://learn.microsoft.com/en-us/nuget/release-notes/nuget-7.0)
- [Auditing package dependencies for security vulnerabilities](https://learn.microsoft.com/en-us/nuget/concepts/auditing-packages)
- [dotnet package update command reference](https://learn.microsoft.com/en-us/dotnet/core/tools/dotnet-package-update)
- [What's new in the SDK and tooling for .NET 10](https://learn.microsoft.com/en-us/dotnet/core/whats-new/dotnet-10/sdk)
- [Package references in project files](https://learn.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files)

