---
title: 'Playwright Workspaces in Azure App Testing: Scaling Browser-Based End-to-End Tests for Government Web Applications'
date: 2026-09-08T20:50:26+00:00
author: Mike Hacker
tags:
- How To
- App Modernization
- Announcements
categories:
- DevOps
summary: A hands-on guide to running Playwright end-to-end and accessibility test suites on Azure's managed cloud browser fleet, including Bicep deployment, Entra ID authentication, CI/CD wiring, and a self-hosted fallback for Azure Government.
draft: false
image_prompt: 'Editorial illustration for a public-sector technology article. Show a clean landscape composition: a single simplified developer workstation or CI runner at lower left, drawn as a solid charcoal block with a few cable-like lines, directing several bright but non-glowing paths toward a broad grid of oversized browser-window silhouettes floating on the right. Each browser window is blank, with no interface details, text, numbers, logos, or charts. In the center, include one large abstract accessibility marker as a simple high-contrast check-shaped path integrated into the flow, not as a badge or icon with lettering. The focal point is the relationship between one local runner and many parallel cloud browsers. Use crisp vector-like editorial illustration, strong silhouettes, generous margins, thumbnail-readable shapes, sunlit white background, charcoal structure, coral accents, teal paths, and soft gray browser panels. Keep all important content away from edges. No visible text, numbers, captions, watermarks, product logos, or branded UI.'
cover_art:
  schemaVersion: 1
  insight: Managed cloud browsers can remove the runtime and infrastructure bottleneck for government end-to-end and accessibility testing, but the test code, identity boundary, cost tuning, and conformance limits still remain the team’s responsibility.
  visualRelationship: A local test runner remains the source of truth while many remote browser windows execute the same checks in parallel, with a separate contained path for environments that must stay inside a government boundary.
  selected: &o0
    subject: local test bench directing a cloud browser fleet
    medium: editorial-illustration
    palette: sunlit white, charcoal, coral, teal, soft gray
    composition: wide landscape with one sturdy workstation shape at lower left sending clean beams to a grid of large browser-window silhouettes across the right; a small accessibility check motif is central but abstract
    rationale: 'The image makes the article’s key distinction visible: tests run from the worker, while scalable browsers live elsewhere and turn slow regression work into parallel execution.'
    prompt: 'Editorial illustration for a public-sector technology article. Show a clean landscape composition: a single simplified developer workstation or CI runner at lower left, drawn as a solid charcoal block with a few cable-like lines, directing several bright but non-glowing paths toward a broad grid of oversized browser-window silhouettes floating on the right. Each browser window is blank, with no interface details, text, numbers, logos, or charts. In the center, include one large abstract accessibility marker as a simple high-contrast check-shaped path integrated into the flow, not as a badge or icon with lettering. The focal point is the relationship between one local runner and many parallel cloud browsers. Use crisp vector-like editorial illustration, strong silhouettes, generous margins, thumbnail-readable shapes, sunlit white background, charcoal structure, coral accents, teal paths, and soft gray browser panels. Keep all important content away from edges. No visible text, numbers, captions, watermarks, product logos, or branded UI.'
    relevance: 5
    clarity: 5
    policyCompliant: true
  concepts:
  - *o0
  - subject: two testing routes from one portable suite
    medium: cut-paper
    palette: warm white, slate, marigold, brick red, muted aqua
    composition: 'central stack of paper browser cards splits into two clear routes: a wide fan of many cards above and a contained row of container-like blocks below'
    rationale: This shows the managed-service path and the Azure Government fallback as practical alternatives using the same Playwright suite, without turning the article into an architecture diagram.
    prompt: 'Cut-paper editorial cover in landscape format. At center, place a thick stack of blank paper browser cards, each with only a plain rectangular silhouette and no markings. From that stack, two simple paper routes diverge: the upper route opens into a broad fan of many evenly spaced blank browser cards, suggesting managed parallel cloud browsers; the lower route becomes a neat row of three sealed container-like paper blocks within a simple boundary line, suggesting a self-hosted fallback kept inside a required environment. Use visible paper fibers, layered shadows, and bold simplified forms. Palette: warm white background, slate gray cards, marigold route strips, brick red boundary accents, muted aqua highlights. Strong central focal point, generous safe margins, no tiny implementation details, no civic buildings, no locks, shields, gears, robots, logos, captions, letters, or numbers.'
    relevance: 5
    clarity: 4
    policyCompliant: true
  - subject: browser test lanes reaching an accessibility finish line
    medium: 3d
    palette: matte charcoal, ivory, amber, cyan-green, pale concrete
    composition: low, wide studio scene with many large blank browser tiles moving along parallel lanes toward one shared raised finish platform; one local runner block anchors the foreground
    rationale: The parallel lanes communicate faster browser-based testing and cost-aware scaling, while the shared finish platform keeps the focus on useful regression evidence rather than total compliance proof.
    prompt: Selective 3D editorial scene, landscape composition, professional and benign. In a clean matte studio environment, show one compact local runner block in the foreground connected to several broad parallel lanes. Along the lanes, large blank browser tiles travel toward a single raised finish platform at the far side. The browser tiles are simple rectangles with empty interiors, no interface chrome, no text, no numbers, no logos. Add a few subtle accessibility-related abstract shapes near the finish platform, such as a clear path and check-like contour, but avoid badges, shields, or literal certification symbols. The focal point is many parallel browser tiles reaching the same endpoint while the local runner stays distinct. Use matte charcoal, ivory, amber lane accents, cyan-green highlights, and pale concrete surfaces. Strong silhouettes, shallow but readable depth, no dramatic blue-purple glow, no brass machinery, no tabletop ornament feel, no visible lettering, captions, watermarks, product branding, or real UI.
    relevance: 4
    clarity: 5
    policyCompliant: true
  review:
    approved: true
    relevant: true
    thumbnailClear: true
    distinctFromRecent: true
    noUnwantedText: true
    noMisleadingDetails: true
    policyCompliant: true
    observations: 'A crisp vector-style landscape illustration shows a dark workstation and laptop at the lower left, teal cable-like paths flowing through a large coral check shape in the center, and a grid of blank pale gray browser-window panels across the right. The background is mostly sunlit white with faint cloud shapes and a very light civic skyline silhouette along the bottom. In the 320px thumbnail, the laptop, central check mark, teal fan-out lines, and browser grid remain readable, while the skyline recedes. Compared with the recent covers, this is visually distinct: flat bright illustration rather than warm dark 3D tabletop scenes with brass, wood, locks, ornaments, and dense miniature objects. No readable text, numbers, logos, watermarks, or product UI claims are visible.'
    feedback: ''
    passes: true
  imageModel: MAI-Image-2.6
  plannerModel: gpt-5.5-low
  reviewerModel: gpt-5.5-high
  generatedAt:
    dateTime: 2026-09-08T21:09:32.8482159
    utcDateTime: 2026-09-08T21:09:32.8482159Z
    localDateTime: 2026-09-08T17:09:32.8482159-04:00
    date: 2026-09-08T00:00:00.0000000
    day: 8
    dayOfWeek: Tuesday
    dayOfYear: 251
    hour: 21
    millisecond: 848
    microsecond: 215
    nanosecond: 900
    minute: 9
    month: 9
    offset: 00:00:00
    totalOffsetMinutes: 0
    second: 32
    ticks: 639244985728482159
    utcTicks: 639244985728482159
    timeOfDay: 21:09:32.8482159
    year: 2026
  attempts: 1
  elapsedSeconds: 156.6014915
image: cover-edfb809f70484b9880421fc5506b3cb7.png
audio: audio.mp3
---

Public sector web teams are under a deadline that is now measured in months, not years. On April 20, 2026, the Federal Register published the Department of Justice's Interim Final Rule extending ADA Title II web accessibility compliance to April 26, 2027 for state and local government entities serving populations of 50,000 or more, and April 26, 2028 for smaller entities and special districts. The technical standard is [WCAG 2.1 Level AA](https://www.ada.gov/resources/2024-03-08-web-rule/).

That deadline lands squarely on the same permitting portals, benefits applications, and tax lookup sites that already carry a backlog of functional defects. Manually regression testing those flows across Chromium, Firefox, and WebKit is not a viable plan. This post walks through **Playwright Workspaces**, the managed cloud browser fleet inside Azure App Testing, and shows how to wire it into a real pipeline, add automated accessibility scanning, and handle the Azure Government availability gap.

## What Playwright Workspaces actually is

[Azure App Testing](https://learn.microsoft.com/en-us/azure/app-testing/overview-what-is-azure-app-testing) is the umbrella service covering two capabilities: Azure Load Testing for performance work, and Playwright Workspaces for functional end-to-end testing. Playwright Workspaces does not run your test code. Your worker processes stay on your machine or CI agent. What moves to the cloud is the expensive part: the browser instances themselves.

That distinction matters for how you tune it, and I will come back to it.

As of the September 2026 revision of the [service limits documentation](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/resource-limits-quotas-capacity), the browser fleet operates in seven Azure regions: East US, West US 3, East Asia, West Europe, Australia East, Japan East, and Switzerland North. Australia East, Japan East, and Switzerland North are the most recent additions to that list. Current default quotas:

| Resource | Limit |
|---|---|
| Workspaces per region per subscription | 2 |
| Parallel workers per workspace | 100 |
| Access tokens per user per workspace | 10 |

The service requires Playwright OSS 1.50 or higher and supports the Playwright test runner and the NUnit runner. Quota increases are requested through the service's GitHub repository, not a standard Azure support ticket.

## Deploying a workspace with Bicep

The portal walkthrough is fine for a proof of concept, but you want this in source control. The resource provider is `Microsoft.LoadTestService/playwrightWorkspaces`, documented in the [Bicep and ARM reference](https://learn.microsoft.com/en-us/azure/templates/microsoft.loadtestservice/playwrightworkspaces).

```bicep
@description('Workspace name: 3-24 characters, alphanumeric and hyphens only.')
@minLength(3)
@maxLength(24)
param workspaceName string

@allowed([
  'eastus'
  'westus3'
  'westeurope'
  'eastasia'
  'australiaeast'
  'japaneast'
  'switzerlandnorth'
])
param location string = 'eastus'

resource pww 'Microsoft.LoadTestService/playwrightWorkspaces@2025-09-01' = {
  name: workspaceName
  location: location
  properties: {
    // Turn off long-lived access tokens. Entra ID only.
    localAuth: 'Disabled'
    // Connect workers to browsers in the nearest region.
    regionalAffinity: 'Enabled'
    reporting: 'Enabled'
  }
  tags: {
    workload: 'citizen-portal-e2e'
  }
}

output workspaceId string = pww.id
```

Deploy and assign RBAC:

```bash
az group create --name rg-e2e-testing --location eastus

az deployment group create \
  --resource-group rg-e2e-testing \
  --template-file playwright-workspace.bicep \
  --parameters workspaceName=pww-citizen-portal location=eastus

WS_ID=$(az deployment group show -g rg-e2e-testing -n playwright-workspace \
  --query properties.outputs.workspaceId.value -o tsv)

az role assignment create \
  --assignee-object-id $CI_PRINCIPAL_OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Playwright Workspace Contributor" \
  --scope $WS_ID
```

The single most important line above is `localAuth: 'Disabled'`. The service supports both Entra ID and access tokens, and Microsoft's own guidance is blunt about it: access tokens "function like long-lived passwords and are more susceptible to being compromised." Access token authentication is disabled by default. Leave it that way. For agencies operating under a zero-trust mandate or CJIS obligations, a static test credential sitting in a pipeline variable group is exactly the kind of finding you do not want in an audit.

Three built-in roles exist, per the [workspace access documentation](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/how-to-manage-workspace-access): Playwright Workspace Reader (view results only), Contributor (run tests), and Owner (run tests plus assign roles). Assign Contributor to your CI identity and Reader to the QA leads who only need dashboard access. Microsoft also recommends managing these through Entra security groups rather than per-user assignments, which avoids brushing up against subscription role assignment limits.

Note that role assignments can take up to an hour to take effect over cached permissions. Budget for that when you are debugging a first-run 403.

## Wiring the service configuration

Scaffold the config with the Microsoft-published package:

```bash
npm init @azure/playwright@latest
```

This generates `playwright.service.config.ts`. The [continuous testing quickstart](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/quickstart-automate-end-to-end-testing) documents the shape:

```typescript
import { defineConfig } from '@playwright/test';
import { createAzurePlaywrightConfig, ServiceOS } from '@azure/playwright';
import { DefaultAzureCredential } from '@azure/identity';
import config from './playwright.config';

export default defineConfig(
  config,
  createAzurePlaywrightConfig(config, {
    exposeNetwork: '<loopback>',
    connectTimeout: 3 * 60 * 1000,
    os: ServiceOS.LINUX,
    credential: new DefaultAzureCredential(),
  })
);
```

`exposeNetwork: '<loopback>'` is worth understanding. It lets a cloud-hosted browser reach `localhost` on the machine running the workers. For a government team testing a portal that is only reachable from inside a corporate network or a private App Service, this means you can run the site locally or on a self-hosted agent behind the firewall and still drive it from managed browsers, without punching a hole in the perimeter.

The regional endpoint goes in `PLAYWRIGHT_SERVICE_URL`, which you copy from the workspace's **Get Started** page. Then:

```bash
az login --tenant <TenantID>
npx playwright test --config=playwright.service.config.ts --workers=20
```

## Adding accessibility scans to the same run

This is where the compliance deadline and the test infrastructure converge. Deque's `@axe-core/playwright` package injects axe-core into every frame and exposes a chainable API. Per the [package README](https://github.com/dequelabs/axe-core-npm/blob/develop/packages/playwright/README.md), the package version tracks the major and minor version of the underlying axe-core engine.

```bash
npm install --save-dev @axe-core/playwright
```

```typescript
import { test, expect } from '@playwright/test';
import { AxeBuilder } from '@axe-core/playwright';

const CRITICAL_PATHS = [
  '/permits/apply',
  '/permits/status',
  '/benefits/eligibility',
  '/payments/checkout',
];

for (const path of CRITICAL_PATHS) {
  test(`WCAG 2.1 AA scan: ${path}`, async ({ page }, testInfo) => {
    await page.goto(path);
    await page.waitForLoadState('networkidle');

    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
      .exclude('#third-party-payment-iframe')
      .analyze();

    await testInfo.attach(`axe-${path.replace(/\//g, '_')}.json`, {
      body: JSON.stringify(results.violations, null, 2),
      contentType: 'application/json',
    });

    expect(results.violations).toEqual([]);
  });
}
```

Because each of these tests gets its own worker and its own cloud browser, a 200-page scan that would serialize for 40 minutes on a build agent finishes in a fraction of that. The `testInfo.attach` call puts the raw violation JSON into the Playwright report, which is what you want when a procurement officer asks for evidence of remediation progress.

Two caveats worth stating plainly. Automated scanning catches roughly a third to a half of WCAG failures. It will not tell you whether your alt text is meaningful or whether your focus order makes sense to a screen reader user. It is a regression net, not a conformance certificate. Second, Section 508 as codified in the [Revised 508 Standards](https://www.access-board.gov/ict/) incorporates WCAG 2.0 Level AA by reference, while the ADA Title II rule sets WCAG 2.1 Level AA. Tag for both, as the snippet above does.

## CI/CD integration

**GitHub Actions**, using OIDC so no secret ever holds a credential:

```yaml
name: E2E and Accessibility
on:
  pull_request:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 60
    steps:
      - uses: actions/checkout@v4
      - uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      - run: npm ci
      - name: Run tests on cloud browsers
        env:
          PLAYWRIGHT_SERVICE_URL: ${{ secrets.PLAYWRIGHT_SERVICE_URL }}
        run: npx playwright test -c playwright.service.config.ts --workers=20
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 10
```

The federated credential is configured on either a Microsoft Entra application or a user-assigned managed identity, trusting tokens issued by your specific repository.

**Azure Pipelines** uses an Azure Resource Manager service connection with workload identity federation:

```yaml
steps:
- task: PowerShell@2
  displayName: Install dependencies
  inputs:
    targetType: inline
    script: npm ci

- task: AzureCLI@2
  displayName: Run Playwright tests
  env:
    PLAYWRIGHT_SERVICE_URL: $(PLAYWRIGHT_SERVICE_URL)
  inputs:
    azureSubscription: sc-playwright-workspace
    scriptType: pscore
    scriptLocation: inlineScript
    inlineScript: |
      npx playwright test -c playwright.service.config.ts --workers=20

- task: PublishPipelineArtifact@1
  condition: always()
  inputs:
    targetPath: playwright-report
    artifact: playwright-report
```

## Tuning parallelism and cost

Billing is based on total test minutes, so parallelism is a direct cost lever. Microsoft's [guidance on optimal configuration](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/concept-determine-optimal-configuration) makes a point that teams consistently miss: completion time hits a floor, after which adding workers only adds cost.

The reason goes back to the architecture. Worker processes still run on your CI agent. Push past roughly 20 workers on a standard two-core hosted runner and the agent itself becomes the bottleneck. You are paying for cloud browsers that sit idle waiting on a starved local process. The fix is to scale the client machine, not the worker count. Move to a larger runner or shard across multiple agents.

A practical sequence: validate locally with at least two workers to shake out parallelism bugs, run against the service at 10 workers, then step to 20, 30, and 50 while measuring wall-clock time. Stop where the curve flattens. Also confirm your target application can absorb the load, because 50 parallel browsers hitting a staging portal generates real traffic.

Leave `regionalAffinity` enabled so workers connect to browsers in the closest region. Test results are collected in the browser region and then transferred to the workspace region. Per the App Testing overview, the service does not store or process customer data outside the region where the workspace is deployed, and all data at rest is encrypted with service-managed keys.

If your build agents sit behind a strict egress firewall, the outbound IP ranges per region are published in the limits documentation. East US is `52.190.15.208/28` and `48.211.3.96/27`; West US 3 is `20.172.9.112/28` and `4.149.23.64/27`.

## Why This Matters for Government

Three reasons this belongs on your roadmap rather than your backlog.

**The compliance clock is real.** WCAG 2.1 Level AA is a regulatory obligation with a date attached, not a design aspiration. Agencies that treat accessibility as a pre-launch audit will discover violations after the code is frozen. Agencies that run axe-core on every pull request treat it as a build break, which costs a developer ten minutes instead of a remediation contract costing six figures.

**Release velocity is a service delivery issue.** When a regression suite takes six hours, teams batch changes into quarterly releases. When it takes twelve minutes, they ship weekly. For a residents' benefits portal, that is the difference between a broken eligibility form being live for a quarter versus a day.

**No test infrastructure to own.** A browser grid is a fleet of VMs with OS patches, browser version drift, and a Selenium-era maintenance burden. Small and mid-sized agency IT shops do not have staff to spare for that. A managed fleet with Entra ID authentication and RBAC removes an entire class of infrastructure while producing a cleaner security posture than the self-hosted alternative.

## The Azure Government caveat, and the fallback

Be clear-eyed here. The documented Playwright Workspaces regions as of September 2026 are all Azure commercial. There are no Azure Government regions on that list. Always confirm current status against [Products available by region](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/) and the [Azure Government comparison guide](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) rather than any blog post, including this one.

For most agencies this is workable, because pre-production test environments frequently run in Azure commercial while production sits in Azure Government. Test data in those environments is synthetic. If your test fixtures contain real resident PII, fix that first regardless of which cloud you are in.

Where you genuinely need the whole pipeline inside Azure Government, self-host on Azure Container Apps Jobs using the official Playwright image. Per the [Playwright Docker guide](https://playwright.dev/docs/docker), the current image is `mcr.microsoft.com/playwright:v1.63.0-noble`, and Microsoft recommends pinning to a specific version so the browser binaries match your project's Playwright version.

```dockerfile
FROM mcr.microsoft.com/playwright:v1.63.0-noble
WORKDIR /tests
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["npx", "playwright", "test", "--shard=${SHARD_INDEX}/${SHARD_TOTAL}"]
```

Then create a scheduled job, using [Container Apps Jobs](https://learn.microsoft.com/en-us/azure/container-apps/jobs) to fan out shards across parallel replicas:

```bash
az containerapp job create \
  --name job-e2e-nightly \
  --resource-group rg-e2e-testing \
  --environment cae-testing \
  --trigger-type Schedule \
  --cron-expression "0 7 * * *" \
  --parallelism 5 \
  --replica-completion-count 5 \
  --replica-timeout 1800 \
  --replica-retry-limit 1 \
  --image myacr.azurecr.us/e2e-tests:1.0.0 \
  --cpu 2.0 --memory 4.0Gi \
  --mi-system-assigned
```

Cron expressions are evaluated in UTC, so `0 7 * * *` is 2 AM Central Standard Time. Use `--parallelism` with `--replica-completion-count` to run shards concurrently, assign a system-assigned managed identity for pulling from Azure Container Registry and reaching Key Vault, and publish the merged HTML report to a storage account.

You lose the managed dashboard and the elastic 100-worker ceiling, but you keep sharded parallel execution, the same test code, and the same axe-core scans, entirely within your authorization boundary. That is a reasonable trade, and it means the work you do today on your Playwright suite is portable whenever the managed service arrives in a region you can use.

## Getting started

Start small. Pick your three highest-traffic resident-facing flows, write Playwright tests for them, add an axe-core scan to each, and wire it into a pull request check. Microsoft's [quickstart](https://learn.microsoft.com/en-us/azure/app-testing/playwright-workspaces/quickstart-run-end-to-end-tests) recommends validating a single test against the service before running a full suite, which is sound advice for managing spend during evaluation.

The hard part was never the browsers. It was building a suite worth running. The managed fleet just removes the excuse that it takes too long.
