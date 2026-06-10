---
title: A Field Guide to .NET 11 Preview 5 for Government Engineering Teams
date: 2026-06-10T16:59:26+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
- Announcements
- Security
categories:
- App Modernization
summary: A hands-on walkthrough of the runtime, ASP.NET Core, EF Core, SDK, and C# changes in .NET 11 Preview 5, with code examples and upgrade guidance for State and Local Government teams running long-lived line-of-business apps.
draft: false
image_prompt: A professional State and Local Government engineering team reviewing .NET modernization plans on secure Azure dashboards, with subtle code, compliance, and cloud infrastructure visuals in a clean Microsoft-inspired style.
image: cover.png
audio: audio.mp3
---

If your agency runs a portfolio of long-lived line-of-business applications - permit systems, case management, payroll, 311 intake - your .NET runtime is not a detail. It is the floor everything else stands on. On June 9, 2026, the .NET team published [.NET 11 Preview 5](https://devblogs.microsoft.com/dotnet/dotnet-11-preview-5/), and this preview is especially relevant to that floor: it includes runtime performance work, an opt-in SDK vulnerability and end-of-life warning, EF Core query and default changes, and C# language features that can move some correctness problems from runtime to compile time.

A planning note first: .NET 10 is the current Long Term Support (LTS) release. It shipped on November 11, 2025, and remains the right target for production workloads today. .NET 11 is still a preview release, and preview releases are not generally supported for production use. Based on the .NET annual release cadence, .NET 11 is expected to reach general availability in November 2026 as a Standard Term Support (STS) release. Use the previews to validate your code and plan your upgrade, not to ship regulated workloads.

## Runtime: performance you do not have to refactor for

The most useful runtime work in any preview is the kind you get without changing application source. Preview 5 delivers several JIT improvements in that category, detailed in the [runtime release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview5/runtime.md).

The JIT now eliminates more redundant bounds checks in span loops. A common SIMD-style pattern - slice off a fixed-width prefix while a length guard holds - now compiles without the extra range-check block:

```csharp
using System.Runtime.Intrinsics;

int Sum(ReadOnlySpan<int> data)
{
    Vector128<int> sum = default;
    while (data.Length >= Vector128<int>.Count)
    {
        sum += Vector128.Create(data);
        data = data.Slice(Vector128<int>.Count);
    }

    int result = Vector128.Sum(sum);
    foreach (int t in data) result += t;
    return result;
}
```

The release notes show the generated code for this method shrinking from 113 bytes to 79 bytes because the JIT proved the slice stays in range across the loop back edge. A related change removes redundant checks for `span.Slice(span.Length - constant)`, so reading the final four bytes of a buffer compiles directly to the load after the initial length guard. These patterns show up in parsers, file format readers, and protocol code - the plumbing inside many line-of-business applications.

The second runtime win is `async` suspension. .NET 11 continues the move to runtime-async, and Preview 5 makes suspension and resumption cheaper for methods optimized by on-stack replacement. The release notes report a suspension-heavy benchmark improving from 6357.1 ms to 457.1 ms once resumption goes directly into optimized code instead of the general-purpose transition path. For request-heavy web APIs full of `await`, reducing this overhead matters at scale.

Garbage collection also gets attention. A new `DOTNET_GCTrimYoungestKeepPercent` switch lets the memory-footprint latency mode retain a configurable share of the youngest generation during trimming. Set these environment variables before process startup:

```powershell
$env:DOTNET_GCLatencyLevel = "0"
$env:DOTNET_GCTrimYoungestKeepPercent = "0xF"
```

A separate compaction fix keeps the heap watermark accurate after relocation. The release notes show a large-pages workload reducing out-of-memory events by 35% over five-minute runs. For agencies packing services densely into App Service plans, containers, or AKS nodes, fewer out-of-memory events can translate into better reliability.

## SDK: a supply-chain warning that belongs in build policy

The SDK change most worth your attention is the new opt-in vulnerability and end-of-life check, described in the [SDK release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview5/sdk.md). NuGet already warns about vulnerable packages; this check covers the resolved .NET SDK version. Enable it in a project file or `Directory.Build.props`:

```xml
<PropertyGroup>
  <CheckSdkVulnerabilities>true</CheckSdkVulnerabilities>
</PropertyGroup>
```

With it enabled, restore refreshes SDK release metadata in the background and caches it under the user's `.dotnet` directory. The build task reads that cache and does not contact the network itself. If no cache is available, the build continues without warnings, so government teams with locked-down build environments should make sure restore has access to the metadata source during a controlled step.

The feature emits independent warnings:

```text
warning NETSDK1236: The current .NET SDK has known vulnerabilities (CVE-...). Update to the latest patch.
warning NETSDK1237: The current .NET SDK is end of life. It will receive no further security updates.
```

Treat these as visibility first, then promote them to build failures through your existing CI policy, for example by using warning-as-error settings for the specific warning codes. That distinction matters: `CheckSdkVulnerabilities` reports warnings by itself; your pipeline decides whether those warnings block a merge or release.

Preview 5 also bundles an MCP Server project template. The release notes show the template being created with an explicit transport option:

```text
dotnet new mcpserver --transport local
```

The template supports local and remote transports. Preview 5 also adds `System.Net.Http.Json` to implicit usings for `net11.0` or later C# projects that use `Microsoft.NET.Sdk` and have implicit usings enabled, aligning console and worker projects with common JSON-over-HTTP scenarios. Container publishing also got stricter: the SDK validates the bearer-token realm returned by a registry authentication challenge, requires an absolute URI, requires HTTPS except for registries configured as insecure, and rejects blocked IP literal forms such as loopback, private, link-local, and unspecified addresses.

## C#: turning some runtime bugs into compile errors

The language features in the [C# release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview5/csharp.md) target correctness, which is where long-lived applications quietly accumulate risk. Closed class hierarchies let the compiler enforce switch exhaustiveness:

```csharp
public closed record class GateState;
public record class Closed : GateState;
public record class Open(float Percent) : GateState;

static string Describe(GateState state) => state switch
{
    Closed => "closed",
    Open(var percent) => $"{percent}% open"
};
```

A closed class is implicitly abstract, and its direct subtypes must be declared in the same assembly. That gives the compiler enough information to check whether a switch expression handles all reachable direct subtypes.

Union declarations bring similar discipline to a fixed set of case types:

```csharp
public record class Dog(string Name);
public record class Cat(int Lives);
public union Pet(Dog, Cat);

static string Describe(Pet pet) => pet switch
{
    Dog(var name) => $"dog: {name}",
    Cat(var lives) => $"cat: {lives}"
};
```

Both features are preview language features, and the surface can change before .NET 11 ships. Preview 5 projects that define closed classes need `System.Runtime.CompilerServices.ClosedAttribute` available to the compilation. Projects that declare unions need `System.Runtime.CompilerServices.UnionAttribute` and the `IUnion` support interface available. Experiment with the model, but do not base a production migration on the exact syntax yet.

## ASP.NET Core: Blazor SSR improves forms and grids

For agencies standardizing on Blazor, the [ASP.NET Core release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview5/aspnetcore.md) close real gaps. Server-side rendered forms now get instant client-side validation without a server round trip, enabled by default for SSR forms that include a `DataAnnotationsValidator`. The .NET model remains the source of truth: the server renders validation metadata, and Blazor JavaScript enforces the rules in the browser. A statically rendered intake form can therefore behave more like an interactive form for the resident filling it out, without requiring an interactive circuit.

QuickGrid also works without interactivity. Sortable headers and the paginator render as enhanced forms that drive URL query-string state, so users can sort, page, refresh, and share links on a statically rendered page.

Validation also gained first-class localization support for Blazor and Minimal APIs, resolving error messages and property names from language-specific RESX files:

```csharp
builder.Services.AddValidation()
    .AddValidationLocalization<ValidationMessages>();
```

For jurisdictions with multilingual public-facing requirements, localized validation that you do not hand-roll on every attribute is a meaningful reduction in code and review burden.

## EF Core: read the breaking changes before you upgrade

The [EF Core release notes](https://github.com/dotnet/core/blob/main/release-notes/11.0/preview/preview5/efcore.md) include one default change that deserves attention: the SQL Server provider now uses compatibility level 160, SQL Server 2022, by default. If your database runs an older compatibility level, pin it explicitly so generated SQL stays within what your database understands:

```csharp
optionsBuilder.UseSqlServer(connectionString,
    sqlServerOptions => sqlServerOptions.UseCompatibilityLevel(150));
```

A new analyzer warning, EF1004, flags `ToAsyncEnumerable()` on an `IQueryable<T>`, which can wrap synchronous enumeration in an `IAsyncEnumerable<T>`. For EF Core queries, switch to `AsAsyncEnumerable()` so the database query runs through EF Core's async query pipeline. Query translation also got cleaner by dropping no-op `CAST` wrappers around value-converted columns. `EnsureCreated`, `EnsureCreatedAsync`, `Migrate`, and `MigrateAsync` now use an execution strategy to retry transient failures. Note the breaking changes too: EF Core 11 removes APIs that were marked obsolete before EF Core 11, so clear those warnings while you are still on .NET 10.

## Why This Matters for Government

State and Local Government code tends to live for a decade or more, and the cost of that longevity is paid in two currencies: security exposure and the slow accumulation of fragile code. .NET 11 Preview 5 spends against both.

The SDK vulnerability and end-of-life warning gives teams a concrete signal they can add to build policy and audit evidence. That does not replace a compliance program, but it can support the kind of repeatable control narrative expected in public-sector reviews. Microsoft documentation describes Azure support for [StateRAMP](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-stateramp), [CJIS](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-cjis), and [IRS Publication 1075](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-irs-1075), and those programs all reward disciplined inventory, patching, encryption, and operational control.

The runtime and JIT improvements can lower compute pressure on the same application code, which matters whether your workload runs in Azure global or [Azure Government](https://learn.microsoft.com/en-us/azure/azure-government/documentation-government-welcome). The C# correctness features point toward safer state machines and document-type handling. The ASP.NET Core and EF Core updates reduce custom code in common places: forms, validation, paging, query translation, and transient database retries.

A practical, low-risk path is straightforward: stay on .NET 10 LTS for production, but stand up a Preview 5 branch in your pipeline now. Install the [.NET 11 SDK](https://dotnet.microsoft.com/download/dotnet/11.0), turn on the SDK vulnerability check, decide whether NETSDK1236 and NETSDK1237 should become blocking warnings in CI, clear EF Core obsolete-API and EF1004 warnings, pin SQL Server compatibility level where needed, and benchmark a representative async-heavy service against the runtime changes. By validating against previews, you arrive at .NET 11 general availability with a tested upgrade path instead of a scramble.

Keep the release notes linked above as the authoritative reference. Preview behavior can change, and government engineering teams should treat each preview as a validation milestone rather than a production target.
