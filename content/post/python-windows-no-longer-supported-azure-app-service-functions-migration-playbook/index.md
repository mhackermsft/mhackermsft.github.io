---
title: 'Python on Windows Is No Longer Supported in Azure App Service and Functions: Your Migration Playbook'
date: 2026-04-06T11:51:09+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
- Announcements
categories:
- App Modernization
summary: A practical migration planning guide for government IT teams running Python workloads on Azure App Service and Functions, covering the shift to Linux-based hosting, upcoming version end-of-support dates, and step-by-step audit and transition guidance.
draft: false
image_prompt: A grand neoclassical government building at golden hour, one half of the marble facade bathed in warm amber light while the other half transitions into cool blue digital cloud patterns dissolving into the sky, a large golden python snake draped over the entrance columns, cinematic low-angle shot with dramatic volumetric light. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
audio: audio.mp3
---

If your agency runs Python web applications or serverless functions on Azure, there is an important platform change you need to know about: **Python on Windows is no longer supported** in Azure App Service, and Azure Functions has never supported Python on Windows. Linux is now the only operating system option for running Python workloads on these services.

This is not a future deprecation notice. The change has already taken effect for App Service, and government IT teams that have not yet migrated are operating on unsupported configurations. This guide provides a practical, step-by-step approach to auditing your environment, understanding the timeline, and executing a smooth transition to Linux-based hosting.

## What Changed and Why

Microsoft's official documentation now states plainly:

> "Linux is the only operating system option for running Python apps in App Service. Python on Windows is no longer supported." - [Configure a Linux Python app for Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/configure-language-python)

For Azure Functions, the [supported languages matrix](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages) confirms that Python runs on Linux only, with no Windows support. This applies to all Python versions, including the currently supported stack: Python 3.10, 3.11, 3.12, and 3.13 (GA), with Python 3.14 in preview.

The reasoning behind this change is straightforward. The Python community and ecosystem have increasingly optimized around Linux environments. Azure's Oryx build system, which handles dependency installation and application packaging, operates natively within Linux containers. By consolidating Python support on Linux, Microsoft delivers better performance, faster cold starts, and more reliable deployments.

## Key Dates to Track

While the Windows retirement for Python is already in effect, government agencies should also be tracking Python version end-of-support dates on Azure:

| Python Version | Support Level | Expected End of Support |
|---|---|---|
| Python 3.10 | GA | October 2026 |
| Python 3.11 | GA | October 2027 |
| Python 3.12 | GA | October 2028 |
| Python 3.13 | GA | October 2029 |
| Python 3.14 | Preview | Pending GA |

Source: [Azure Functions Supported Languages](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages)

Additionally, the **Linux Consumption plan for Azure Functions will be retired after September 30, 2028**. Microsoft recommends migrating to the [Flex Consumption plan](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan) before that date for continued serverless Python hosting.

**Important for plan selection:** As of September 30, 2025, the Linux Consumption plan no longer receives new language stack support. The last supported Python version on the Linux Consumption plan is **Python 3.12**. Agencies that need Python 3.13 or later must use the Flex Consumption plan or another hosting option such as the Premium or Dedicated plans.

## Why This Matters for Government

Government agencies often operate with longer procurement and change management cycles than private-sector organizations. A platform change that the private sector might address in weeks can take months in a government IT environment due to security reviews, change advisory boards, and compliance documentation requirements.

Here is why this deserves immediate attention:

- **Compliance risk**: Running applications on unsupported platform configurations means those workloads are not eligible for security patches or Microsoft support. For agencies subject to NIST, StateRAMP, or internal security frameworks, this creates an audit finding waiting to happen.
- **Security exposure**: Unsupported runtime configurations do not receive security updates. In a threat landscape where government organizations are increasingly targeted, this is unacceptable.
- **Azure Government parity**: Both Azure Commercial and [Azure Government](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) support App Service and Azure Functions on Linux. The migration path is consistent across both cloud environments, but agencies should verify that their specific Azure Government region supports the Linux App Service SKUs they need.
- **Budget planning**: Migrating to Linux App Service plans may involve changes to SKU selection and pricing. Agencies should model costs early to align with fiscal year budgets.

## Step 1: Audit Your Python Workloads

Before you can migrate, you need to know what you have. Use the Azure CLI or Azure Resource Graph to identify all App Service and Function apps running Python on Windows.

**Find App Service apps and their OS configuration:**

```bash
az webapp list --query "[?kind=='app'].{Name:name, ResourceGroup:resourceGroup, OS:kind}" -o table
```

**Check the runtime stack for a specific app:**

```bash
az webapp config show --resource-group <resource-group> --name <app-name> --query linuxFxVersion
```

If `linuxFxVersion` is empty and the app is configured with a Python runtime, the app is likely running on Windows and must be migrated.

**For Azure Functions, check the worker runtime and OS:**

```bash
az functionapp list --query "[?siteConfig.linuxFxVersion==null].{Name:name, ResourceGroup:resourceGroup}" -o table
```

Document each application with its current Python version, dependencies, connected data sources, and any custom configuration. This inventory becomes your migration project plan.

## Step 2: Test Locally on Linux

Before deploying to Azure, validate that your application runs correctly on Linux. The most common issues during Windows-to-Linux migration include:

- **File path separators**: Code using backslashes (`\\`) in file paths will fail on Linux. Use `os.path.join()` or `pathlib.Path` instead.
- **Case-sensitive file systems**: Linux file systems are case-sensitive. A reference to `Config.json` will not find `config.json`.
- **Windows-specific dependencies**: Some Python packages include compiled C extensions that are Windows-only. Check your `requirements.txt` for platform-specific packages.
- **Environment variables**: Windows App Service and Linux App Service handle environment variables identically through App Settings, but verify all values carry over during migration.

For local testing, consider using [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/) or Docker with a Linux-based Python image to simulate the target environment.

## Step 3: Create Your Linux App Service or Function App

For App Service web apps, creating a new Linux-based Python app is straightforward with the Azure CLI:

```bash
az webapp up --runtime PYTHON:3.12 --sku B1 --logs --resource-group <resource-group> --name <app-name>
```

This command creates the resource group, App Service plan (Linux), and deploys your application in a single step. See the [Python quickstart for App Service](https://learn.microsoft.com/en-us/azure/app-service/quickstart-python) for detailed instructions.

For Azure Functions, Microsoft recommends the **[Flex Consumption plan](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan)** for new serverless Python workloads. The traditional Linux Consumption plan no longer supports Python versions newer than 3.12 and will be fully retired after September 30, 2028. Flex Consumption offers virtual network integration, per-function scaling, configurable instance sizes, and support for the latest Python versions, all on a pay-for-what-you-use model.

If you have an existing workload that must remain on the Consumption plan temporarily, you can create a Linux-based function app with Python 3.12:

```bash
az functionapp create --resource-group <resource-group> --consumption-plan-location <region> \
  --runtime python --runtime-version 3.12 --functions-version 4 \
  --name <function-app-name> --storage-account <storage-account> --os-type Linux
```

However, plan your migration path to Flex Consumption to ensure access to newer Python versions and continued platform support.

## Step 4: Migrate Configuration and Secrets

Transfer all application settings, connection strings, and managed identity assignments from the existing app to the new Linux-based app:

```bash
# Export settings from old app
az webapp config appsettings list --resource-group <old-rg> --name <old-app> -o json > app-settings.json

# Import settings to new app
az webapp config appsettings set --resource-group <new-rg> --name <new-app> --settings @app-settings.json
```

Verify that any references to Azure Key Vault, managed identities, or virtual network integrations are re-established on the new resource.

## Step 5: Use Deployment Slots for Zero-Downtime Migration

Azure App Service [deployment slots](https://learn.microsoft.com/en-us/azure/app-service/deploy-staging-slots) allow you to deploy your migrated application to a staging environment, validate it thoroughly, and then swap it into production with minimal downtime:

1. Create a staging slot on your new Linux App Service.
2. Deploy your application to the staging slot.
3. Run integration and smoke tests against the staging URL.
4. Swap the staging slot into production.

For Azure Functions, the [update language versions guide](https://learn.microsoft.com/en-us/azure/azure-functions/update-language-versions) recommends the same slot-based approach to minimize risk during runtime transitions.

## Step 6: Update CI/CD Pipelines

Your deployment pipelines likely reference the old Windows-based app. Update them to target the new Linux resource:

- Update the target app name and resource group in your GitHub Actions workflow or Azure Pipelines YAML.
- If you were using `zip deploy`, enable build automation by setting `SCM_DO_BUILD_DURING_DEPLOYMENT=1` on the new Linux app. This triggers the Oryx build system to install dependencies from your `requirements.txt`, `pyproject.toml`, or `setup.py` automatically.
- Verify that your pipeline uses `az webapp deploy` or `az functionapp deployment source config-zip` targeting the correct Linux app.

## Azure Government Considerations

Agencies using Azure Government should note the following:

- App Service and Azure Functions are available in Azure Government with Linux support. The service endpoints differ (e.g., `azurewebsites.us` instead of `azurewebsites.net`).
- Verify region availability for your desired App Service plan SKU in your Azure Government subscription.
- The Flex Consumption plan may have different regional availability in Azure Government compared to Azure Commercial. Check the [Azure products by region](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/) page, filtering for Azure Government regions, to confirm current availability.
- CI/CD pipelines targeting Azure Government must authenticate against the `AzureUSGovernment` cloud environment.

## Build Your Own Container as a Fallback

If your application has dependencies that are difficult to resolve on the built-in Linux Python stack, you can build a custom container image and deploy it to App Service. Microsoft's documentation describes this approach in the [custom container tutorial](https://learn.microsoft.com/en-us/azure/app-service/tutorial-custom-container?pivots=container-linux). This gives you full control over the operating system, installed packages, and Python version.

## Recommended Timeline

Here is a suggested phased approach for government agencies:

1. **Now**: Inventory all Python workloads on App Service and Functions. Identify any still running on Windows.
2. **Within 30 days**: Establish a Linux-based test environment and validate each application.
3. **Within 90 days**: Migrate non-production workloads and update CI/CD pipelines.
4. **Within 180 days**: Complete production migrations with deployment slot swaps.
5. **Ongoing**: Track Python version end-of-support dates and plan version upgrades at least 6 months in advance.

## Resources

- [Configure a Linux Python app for Azure App Service](https://learn.microsoft.com/en-us/azure/app-service/configure-language-python)
- [Azure Functions Python Developer Guide](https://learn.microsoft.com/en-us/azure/azure-functions/functions-reference-python)
- [Azure Functions Supported Languages](https://learn.microsoft.com/en-us/azure/azure-functions/supported-languages)
- [Update Language Stack Versions in Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/update-language-versions)
- [Flex Consumption Plan Overview](https://learn.microsoft.com/en-us/azure/azure-functions/flex-consumption-plan)
- [App Service Language Runtime Support Policy](https://learn.microsoft.com/en-us/azure/app-service/language-support-policy)
- [Azure Government Comparison](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure)
- [Python Quickstart for App Service](https://learn.microsoft.com/en-us/azure/app-service/quickstart-python)
- [Azure Products by Region](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/)
