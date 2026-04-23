---
title: 'Azure Developer CLI (azd) Polyglot Hooks: Automating Government Deployment Workflows in Any Language'
date: 2026-04-23T12:31:19+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
- Announcements
categories:
- Azure Developer Tools
summary: The Azure Developer CLI now supports deployment hooks in Python, JavaScript, TypeScript, and .NET, letting government teams automate cloud workflows in their preferred language instead of being locked into Bash or PowerShell.
draft: false
image_prompt: 'A massive workbench viewed from above, covered in tools from different trades: a brass compass, a set of interlocking steel gears, a glass prism refracting light into colored beams, a coiled copper cable, and a heavy iron skeleton key, all arranged in a radial pattern on dark oiled wood under warm directional spotlight casting long dramatic shadows. No text, letters, numbers, or writing anywhere in the image.'
image: cover.png
audio: audio.mp3
---

Government IT teams rarely standardize on a single programming language. Your infrastructure team writes PowerShell, your data engineers prefer Python, your web developers think in TypeScript, and your line-of-business app team builds everything in .NET. Until now, Azure Developer CLI (`azd`) hooks forced everyone into the same box: Bash or PowerShell. That changed with the April 2026 releases of azd 1.23.15 and 1.24.0, which introduced **polyglot hook support** for Python, JavaScript, TypeScript, and C#/.NET.

This post walks through the new hook system with concrete examples relevant to government deployment workflows, including compliance checks, environment validation, and post-deployment verification.

## What Are azd Hooks?

The Azure Developer CLI uses a lifecycle-based hook system that lets you execute custom scripts before and after key deployment commands. Hooks follow a `pre`/`post` naming convention tied to `azd` commands:

| Hook | Trigger |
|---|---|
| `preprovision` / `postprovision` | Before/after Azure resources are created |
| `predeploy` / `postdeploy` | Before/after application code is deployed |
| `preup` / `postup` | Before/after the combined provision + deploy pipeline |
| `predown` / `postdown` | Before/after resources are torn down |
| `prerestore` / `postrestore` | Before/after package dependencies are restored |

Hooks are registered in your `azure.yaml` file at the project root or scoped to individual services. Previously, you could only set `shell: sh` or `shell: pwsh`. Now, azd **auto-detects the language** from the file extension of your hook script, no shell declaration needed.

For the full hook reference, see the [azd extensibility documentation](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/azd-extensibility).

## What Changed: Language Support Across Two Releases

### azd 1.23.15 (April 10, 2026)

- **Python hooks**: Point a hook's `run:` field at a `.py` file and azd auto-detects it. When a `requirements.txt` or `pyproject.toml` is present in the hook directory, azd automatically creates a virtual environment and installs dependencies before execution.
- **JavaScript hooks**: `.js` files are auto-detected and executed with `node`. When a `package.json` is present, azd runs `npm install` automatically.
- **TypeScript hooks**: `.ts` files execute via `npx tsx` with zero compile step required. Same `package.json` auto-install behavior as JavaScript.

### azd 1.24.0 (April 14, 2026)

- **C#/.NET hooks**: `.cs` files are auto-detected and executed using `dotnet run`. azd discovers `.csproj` files via walk-up search and supports single-file C# scripts on .NET 10+.
- **Executor-specific `config:` block**: Fine-grained control per language, including `packageManager` for JS/TS hooks (npm, pnpm, or yarn), `virtualEnvName` for Python hooks, and `configuration`/`framework` for .NET hooks.

Sources: [azd 1.23.15 release notes](https://github.com/Azure/azure-dev/releases/tag/azure-dev-cli_1.23.15), [azd 1.24.0 release notes](https://github.com/Azure/azure-dev/releases/tag/azure-dev-cli_1.24.0)

## Hands-On: Government Deployment Hooks in Four Languages

Let's build a realistic `azure.yaml` configuration for a government application that uses hooks in every supported language.

### The Scenario

Your agency deploys a citizen services portal to Azure. Before provisioning, you need to validate that the target subscription meets your compliance requirements. After provisioning, you seed a database. After deployment, you run smoke tests and generate an audit report.

### Project Structure

```
├── azure.yaml
├── hooks/
│   ├── validate-subscription.py      # Python: pre-provision compliance check
│   ├── requirements.txt
│   ├── seed-database.cs               # .NET: post-provision data seeding
│   ├── seed-database.csproj
│   ├── smoke-tests.ts                 # TypeScript: post-deploy verification
│   ├── package.json
│   └── generate-audit-report.js       # JavaScript: post-up audit trail
├── infra/
│   └── main.bicep
└── src/
    └── api/
```

### azure.yaml Configuration

```yaml
name: citizen-services-portal
metadata:
  template: citizen-services-portal@1.0.0

hooks:
  preprovision:
    run: ./hooks/validate-subscription.py
    config:
      virtualEnvName: compliance_venv

  postprovision:
    run: ./hooks/seed-database.cs
    config:
      configuration: Release
      framework: net8.0

  postdeploy:
    run: ./hooks/smoke-tests.ts
    config:
      packageManager: npm

  postup:
    run: ./hooks/generate-audit-report.js

services:
  api:
    project: ./src/api
    language: csharp
    host: containerapp
```

Notice there is no `shell:` declaration on any hook. azd infers the executor from the file extension: `.py` runs Python, `.cs` runs .NET, `.ts` runs TypeScript via `npx tsx`, and `.js` runs JavaScript via `node`.

### Hook 1: Python Pre-Provision Compliance Check

**hooks/validate-subscription.py**

```python
"""Pre-provision hook: validate subscription compliance requirements."""
import os
import subprocess
import json
import sys

def get_azd_env(key: str) -> str:
    """Retrieve an azd environment variable."""
    result = subprocess.run(
        ["azd", "env", "get-value", key],
        capture_output=True, text=True
    )
    return result.stdout.strip()

def check_resource_providers(subscription_id: str, required: list[str]) -> list[str]:
    """Verify required resource providers are registered."""
    result = subprocess.run(
        ["az", "provider", "list", "--subscription", subscription_id,
         "--query", "[?registrationState=='Registered'].namespace",
         "-o", "json"],
        capture_output=True, text=True
    )
    registered = json.loads(result.stdout)
    return [p for p in required if p not in registered]

def check_allowed_regions(location: str) -> bool:
    """Validate deployment targets an approved government region."""
    approved_regions = [
        "usgovvirginia", "usgovarizona", "usgovtexas",
        "eastus", "eastus2", "centralus", "northcentralus"
    ]
    return location.lower() in approved_regions

def main():
    subscription_id = get_azd_env("AZURE_SUBSCRIPTION_ID")
    location = get_azd_env("AZURE_LOCATION")

    print(f"Validating subscription {subscription_id} in {location}...")

    # Check region compliance
    if not check_allowed_regions(location):
        print(f"ERROR: Region '{location}' is not in the approved list.")
        sys.exit(1)
    print(f"Region '{location}' is approved.")

    # Check required resource providers
    required_providers = [
        "Microsoft.App",
        "Microsoft.ContainerRegistry",
        "Microsoft.KeyVault",
        "Microsoft.OperationalInsights"
    ]
    missing = check_resource_providers(subscription_id, required_providers)
    if missing:
        print(f"ERROR: Missing resource providers: {', '.join(missing)}")
        print("Register them with: az provider register --namespace <provider>")
        sys.exit(1)
    print("All required resource providers are registered.")

    print("Subscription compliance validation passed.")

if __name__ == "__main__":
    main()
```

**hooks/requirements.txt**

```
# No external dependencies needed for this hook.
# azd will still create the venv automatically.
```

azd automatically creates the `compliance_venv` virtual environment (using the name from the `config:` block) and installs any dependencies listed in `requirements.txt` before running the script.

### Hook 2: .NET Post-Provision Database Seeding

**hooks/seed-database.cs**

```csharp
using System;
using System.Diagnostics;
using System.Net.Http;
using System.Net.Http.Json;
using System.Threading.Tasks;

var connectionString = Environment.GetEnvironmentVariable("AZURE_COSMOS_CONNECTION_STRING");
if (string.IsNullOrEmpty(connectionString))
{
    Console.WriteLine("AZURE_COSMOS_CONNECTION_STRING not set. Skipping seed.");
    return;
}

Console.WriteLine("Seeding reference data after provisioning...");

// azd environment variables are automatically available
var envName = Environment.GetEnvironmentVariable("AZURE_ENV_NAME");
Console.WriteLine($"Environment: {envName}");

// Seed logic using your preferred .NET libraries
Console.WriteLine("Inserting department reference data...");
Console.WriteLine("Inserting service category lookup tables...");
Console.WriteLine("Database seeding complete.");
```

With the `config:` block set to `configuration: Release` and `framework: net8.0`, azd runs `dotnet restore`, `dotnet build -c Release -f net8.0`, and then `dotnet run --no-build --project <discovered-csproj>`. For teams on .NET 10+, single-file `.cs` scripts work without a `.csproj` at all.

### Hook 3: TypeScript Post-Deploy Smoke Tests

**hooks/smoke-tests.ts**

```typescript
import { execSync } from "child_process";

interface HealthCheckResult {
  service: string;
  status: "healthy" | "unhealthy";
  responseTimeMs: number;
}

function getAzdEnv(key: string): string {
  return execSync(`azd env get-value ${key}`, { encoding: "utf-8" }).trim();
}

async function checkEndpoint(name: string, url: string): Promise<HealthCheckResult> {
  const start = Date.now();
  try {
    const response = await fetch(`${url}/health`);
    return {
      service: name,
      status: response.ok ? "healthy" : "unhealthy",
      responseTimeMs: Date.now() - start,
    };
  } catch {
    return { service: name, status: "unhealthy", responseTimeMs: Date.now() - start };
  }
}

async function main(): Promise<void> {
  const apiEndpoint = getAzdEnv("SERVICE_API_ENDPOINT_URL");
  console.log(`Running smoke tests against ${apiEndpoint}...`);

  const results: HealthCheckResult[] = [
    await checkEndpoint("API Health", apiEndpoint),
    await checkEndpoint("API Auth", `${apiEndpoint}/api`),
  ];

  console.table(results);

  const failures = results.filter((r) => r.status === "unhealthy");
  if (failures.length > 0) {
    console.error(`${failures.length} endpoint(s) failed smoke tests.`);
    process.exit(1);
  }
  console.log("All smoke tests passed.");
}

main();
```

TypeScript hooks execute through `npx tsx`, which means **zero compilation step**. You write `.ts`, azd runs `.ts`. The `package.json` in the hooks directory can declare dependencies like any Node project, and azd runs `npm install` (or `pnpm`/`yarn` if configured) during the `Prepare` phase.

### Hook 4: JavaScript Post-Up Audit Report

**hooks/generate-audit-report.js**

```javascript
const { execSync } = require("child_process");
const fs = require("fs");

function getAzdEnv(key) {
  return execSync(`azd env get-value ${key}`, { encoding: "utf-8" }).trim();
}

const report = {
  timestamp: new Date().toISOString(),
  environment: getAzdEnv("AZURE_ENV_NAME"),
  subscription: getAzdEnv("AZURE_SUBSCRIPTION_ID"),
  location: getAzdEnv("AZURE_LOCATION"),
  resourceGroup: getAzdEnv("AZURE_RESOURCE_GROUP"),
  deployedBy: execSync("az account show --query user.name -o tsv", {
    encoding: "utf-8",
  }).trim(),
  status: "completed",
};

const filename = `audit-${report.environment}-${Date.now()}.json`;
fs.writeFileSync(filename, JSON.stringify(report, null, 2));
console.log(`Audit report written to ${filename}`);
console.log(JSON.stringify(report, null, 2));
```

## Running and Testing Hooks Independently

azd 1.24.0 also supports running hooks in isolation for testing, which is especially useful when building out compliance automation:

```bash
# Test the pre-provision compliance check without actually provisioning
azd hooks run preprovision

# Test a service-specific hook
azd hooks run postdeploy --service api

# Test against a specific environment
azd hooks run postup -e staging
```

This makes it straightforward to iterate on hook logic in a development environment before promoting to production.

## Platform-Specific Overrides

Government teams often have mixed Windows and Linux development environments. Polyglot hooks support platform-specific overrides:

```yaml
hooks:
  preprovision:
    windows:
      run: ./hooks/validate-subscription.py
      config:
        virtualEnvName: .venv
    posix:
      run: ./hooks/validate-subscription.py
      config:
        virtualEnvName: .venv
```

## Integrating with CI/CD Pipelines

Hooks execute in CI/CD pipelines exactly as they do locally. When you run `azd pipeline config` to set up [GitHub Actions or Azure Pipelines](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/configure-devops-pipeline), your hooks run as part of the `azd up` or `azd deploy` steps in the pipeline. This means your compliance checks, smoke tests, and audit reporting run automatically on every deployment, not just when a developer remembers to run them.

## Why This Matters for Government

**Meet teams where they are.** Government IT organizations are rarely single-language shops. Your security team might maintain compliance scripts in Python, your platform team automates in PowerShell, and your application developers build in .NET or TypeScript. Polyglot hooks eliminate the friction of translating automation logic into a language the tool supports.

**Codify compliance as deployment gates.** The `preprovision` hook pattern shown above turns compliance validation into an automated gate that runs before any resources are created. Region restrictions, resource provider checks, naming convention enforcement, and policy validation can all be expressed in the language your compliance team already knows.

**Auditable deployment trails.** Government agencies face audit requirements that commercial organizations do not. The `postup` hook pattern generates structured audit records for every deployment, capturing who deployed, when, where, and to which environment, all without manual documentation.

**Consistent across environments.** Whether your team deploys to Azure Commercial (East US, Central US) or [Azure Government](https://learn.microsoft.com/en-us/azure/azure-government/documentation-government-welcome) (US Gov Virginia, US Gov Arizona), azd hooks work identically. The CLI is a client-side tool, so there are no cloud-side feature availability concerns.

**Lower the barrier to infrastructure automation.** Many government developers are not shell scripting experts. Letting them write deployment hooks in Python, TypeScript, or C# means more teams can participate in building deployment automation, reducing bottlenecks on the one person who knows Bash.

## Getting Started

1. **Install or update azd** to version 1.24.0 or later:
   ```
   winget upgrade microsoft.azd
   ```
   Or on Linux/macOS:
   ```bash
   curl -fsSL https://aka.ms/install-azd.sh | bash
   ```

2. **Add hooks to your `azure.yaml`** using the examples above as templates.

3. **Test hooks independently** with `azd hooks run <hook-name>` before running full deployments.

4. **Review the official documentation** at [Customize your azd workflows using hooks](https://learn.microsoft.com/en-us/azure/developer/azure-developer-cli/azd-extensibility).

Polyglot hooks are a small feature with outsized impact. They turn `azd` from a deployment tool into a deployment platform where your entire team, regardless of language preference, can contribute to automated, compliant, auditable cloud deployments.
