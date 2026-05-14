---
title: 'Azure Container Apps Express: Fast-Track Your Government Container Deployments'
date: 2026-05-14T17:12:46+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
- Announcements
categories:
- App Modernization
summary: Azure Container Apps Express is now in public preview, offering a radically simplified portal-first deployment model that gets government web apps and APIs running in minutes with scale-to-zero economics and zero infrastructure management.
draft: false
image_prompt: A modern government building facade with a digital overlay showing containers being deployed rapidly through a streamlined pipeline, with Azure blue accents and a clean, minimal aesthetic conveying speed and simplicity
image: cover.png
audio: audio.mp3
---

Government IT teams are under constant pressure to deliver digital services faster while keeping budgets lean. Azure Container Apps has already simplified container hosting by abstracting away Kubernetes, but the new **Azure Container Apps Express** deployment model - now in public preview - takes that simplification even further. Express strips the experience down to the absolute minimum: deploy a container image, get a URL, and let the platform handle everything else.

This post walks through what Express is, how it differs from standard Container Apps, and provides hands-on deployment examples you can run today.

## What Is Container Apps Express?

Container Apps Express is a new deployment model within Azure Container Apps designed for **HTTP-first workloads** where speed and simplicity matter most. Rather than creating and configuring a Container Apps environment, selecting workload profiles, and tuning scaling parameters, Express applies opinionated defaults and provisions the underlying infrastructure automatically.

Key capabilities include:

- **Deploy in minutes** with no infrastructure tuning required
- **Scale from zero to hyperscale** automatically based on incoming HTTP traffic
- **Scale-to-zero economics** so your app scales down when idle and back up on demand
- **Optimized cold start** so your app responds quickly after scaling from zero
- **Minimal configuration surface** with sensible production-ready defaults

Express is ideal for REST APIs, SaaS frontends, AI application gateways, internal web tools, and rapid prototyping scenarios. It is *not* designed for TCP-based services, GPU workloads, background jobs, or microservice architectures that need Dapr service discovery.

For the full overview, see the [Azure Container Apps Express documentation](https://learn.microsoft.com/en-us/azure/container-apps/express-overview).

## Express vs. Standard Container Apps: When to Use Which

Express is not a replacement for standard Container Apps - it is a streamlined on-ramp for a specific class of workloads. Here is how they compare:

| Capability | Express | Standard Container Apps |
|---|---|---|
| Environment management | Automatic (portal) / Manual (CLI) | Manual |
| Scale to zero | Yes | Yes |
| HTTP ingress | Yes | Yes |
| TCP ingress | No | Yes |
| VNet integration | No | Yes |
| Managed identity | No (preview limitation) | Yes |
| Dapr integration | No | Yes |
| Custom domains | No (preview limitation) | Yes |
| KEDA-based autoscaling | No | Yes |
| Secrets / Key Vault | No (preview limitation) | Yes |
| GPU support | No | Yes |
| Multi-revision traffic splitting | No | Yes |

The simplest decision framework: if your workload is a web app or API that communicates over HTTP and you want the fastest path to production, start with Express. If you need VNet isolation, managed identity, custom domains, or event-driven scaling, use standard Container Apps.

## Hands-On: Deploying with the Azure CLI

The Express deployment model delivers its streamlined experience primarily through the dedicated portal at [containerapps.azure.com](https://containerapps.azure.com/). When using the CLI, Express still requires you to create an environment manually, following the same workflow as standard Container Apps. The commands below deploy a container app using the standard CLI workflow with a minimal configuration that mirrors the Express philosophy of simplicity.

First, ensure your CLI is up to date and the Container Apps extension is installed with preview features enabled:

```bash
az upgrade
az extension add --name containerapp --upgrade --allow-preview true

# Register required providers
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
```

Next, set your variables and create a resource group:

```bash
RESOURCE_GROUP="rg-express-demo"
LOCATION="westcentralus"  # Express preview region
APP_NAME="gov-api-express"
ENVIRONMENT_NAME="express-env"

az group create --name $RESOURCE_GROUP --location $LOCATION
```

Create a lightweight Container Apps environment and deploy your app:

```bash
# Create the environment
az containerapp env create \
  --name $ENVIRONMENT_NAME \
  --resource-group $RESOURCE_GROUP \
  --location $LOCATION

# Deploy a container app with minimal configuration
az containerapp create \
  --name $APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --environment $ENVIRONMENT_NAME \
  --image mcr.microsoft.com/k8se/quickstart:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 10 \
  --query properties.configuration.ingress.fqdn \
  --output tsv
```

That is it. The CLI returns your application's fully qualified domain name. Your app is live, scaling from zero replicas when idle and automatically adding replicas as HTTP traffic arrives.

For the true Express portal experience, visit [containerapps.azure.com](https://containerapps.azure.com/) where the platform creates the environment automatically and applies opinionated defaults with no CLI steps required.

## Scaling Configuration Deep Dive

While Express handles scaling automatically with opinionated defaults, understanding how standard Container Apps scaling works helps you make informed decisions when you need more control. Standard Container Apps use KEDA (Kubernetes Event-driven Autoscaling) under the hood, and the scaling configuration is declarative.

Here is a CLI example that configures HTTP-based autoscaling for a government API that needs to handle variable citizen traffic:

```bash
az containerapp create \
  --name citizen-portal-api \
  --resource-group $RESOURCE_GROUP \
  --environment $ENVIRONMENT_NAME \
  --image myregistry.azurecr.io/citizen-api:v1.2 \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 20 \
  --scale-rule-name http-scaling \
  --scale-rule-http-concurrency 50
```

This configuration keeps at least one replica warm at all times (no cold starts for citizens), scales up when concurrent requests per replica exceed 50, and caps at 20 replicas. The platform calculates the number of concurrent requests every 15 seconds as the number of requests in the past 15 seconds divided by 15.

## Bicep Template: Production-Ready Deployment

For government organizations practicing Infrastructure as Code, here is a Bicep template that deploys a standard Container Apps environment and application with HTTP scaling rules. This is the pattern you would adopt as your workload matures beyond Express and requires features like custom scaling rules, VNet integration, or managed identity:

```bicep
@description('Location for all resources')
param location string = 'westcentralus'

@description('Container image to deploy')
param containerImage string = 'mcr.microsoft.com/k8se/quickstart:latest'

@description('Target port for the container')
param targetPort int = 80

@description('Maximum number of replicas')
param maxReplicas int = 10

resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'law-containerapps'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

resource containerAppEnv 'Microsoft.App/managedEnvironments@2025-02-02-preview' = {
  name: 'env-gov-apps'
  location: location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
  }
}

resource containerApp 'Microsoft.App/containerApps@2025-02-02-preview' = {
  name: 'gov-web-app'
  location: location
  properties: {
    environmentId: containerAppEnv.id
    configuration: {
      activeRevisionsMode: 'Single'
      ingress: {
        external: true
        targetPort: targetPort
        transport: 'http'
      }
    }
    template: {
      containers: [
        {
          name: 'gov-web-app'
          image: containerImage
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
        }
      ]
      scale: {
        minReplicas: 0
        maxReplicas: maxReplicas
        rules: [
          {
            name: 'http-scaling-rule'
            http: {
              metadata: {
                concurrentRequests: '50'
              }
            }
          }
        ]
      }
    }
  }
}

output appUrl string = 'https://${containerApp.properties.configuration.ingress.fqdn}'
```

Deploy this template with a single command:

```bash
az deployment group create \
  --resource-group rg-express-demo \
  --template-file main.bicep \
  --parameters containerImage='myregistry.azurecr.io/citizen-api:v1.2' \
               targetPort=8080 \
               maxReplicas=20
```

This template creates the Log Analytics workspace, the Container Apps environment, and the application in a single declarative deployment - perfect for CI/CD pipelines and repeatable government deployments.

## Government Deployment Patterns

Here are three practical patterns for state and local government teams considering Container Apps Express:

**Pattern 1: Citizen-Facing API Gateway.** Deploy a lightweight API that fronts existing backend services. Express's scale-to-zero means you are not paying for idle capacity during off-hours, while automatic scaling handles spikes during tax season, permit application deadlines, or public comment periods.

**Pattern 2: Internal Developer Tools.** Government dev teams can rapidly deploy internal tools, documentation sites, or admin interfaces without filing infrastructure requests. Express's opinionated defaults mean junior developers can ship without deep platform expertise.

**Pattern 3: AI Application Frontend.** With optimized cold start and automatic scaling, Express is well-suited for hosting the web frontend of AI-powered applications. Pair it with Azure OpenAI Service for a chatbot or document analysis tool that scales with demand and costs nothing when idle.

## Preview Limitations to Watch

As a public preview, Express has important limitations that government teams should be aware of:

- **No managed identity support** - you cannot authenticate to other Azure services using managed identity yet. This is a significant gap for production government workloads.
- **No VNet integration** - Express apps run on shared infrastructure without custom network isolation.
- **No secrets or Key Vault integration** - sensitive configuration values cannot be securely stored.
- **No custom domains** - apps use platform-generated URLs only.
- **Limited region availability** - currently available only in **West Central US** and **East Asia**.
- **No health probes, CORS, or IP restrictions** - features that many production apps require.
- **Billing not yet enabled** - the Express supported features table lists billing as not yet supported during the preview. Check the [Express overview documentation](https://learn.microsoft.com/en-us/azure/container-apps/express-overview) for the latest billing status before planning cost projections.

These limitations make Express best suited for development, prototyping, and non-sensitive internal workloads during the preview period. Plan to migrate to standard Container Apps for production government workloads that require identity, networking, and compliance features.

For the full supported features matrix, see the [Express overview documentation](https://learn.microsoft.com/en-us/azure/container-apps/express-overview).

## Why This Matters for Government

State and local government organizations often struggle with the tension between security requirements and development velocity. Container Apps Express directly addresses the velocity side of this equation:

- **Budget efficiency**: Scale-to-zero economics means agencies can avoid paying for idle compute capacity - critical for departments with tight IT budgets and seasonal traffic patterns. Verify the current billing status on the [Express overview page](https://learn.microsoft.com/en-us/azure/container-apps/express-overview), as billing features are still being rolled out during preview.
- **Reduced time to deployment**: Getting from container image to running URL in minutes (rather than days of infrastructure provisioning) accelerates pilot programs and proof-of-concept delivery.
- **Lower skill barrier**: Express removes the need for Kubernetes expertise, networking configuration, and scaling tuning. Government teams with limited DevOps staffing can still ship containerized applications.
- **Faster modernization cycles**: Legacy application modernization projects can use Express to quickly stand up new API layers or web frontends while backend systems are incrementally refactored.
- **AI readiness**: As government agencies explore AI-powered citizen services, Express provides a hosting model for AI application frontends that need elastic scaling.

For production workloads requiring network isolation and compliance controls, standard Azure Container Apps with VNet integration and managed identity remains the appropriate choice. Express is the rapid prototyping and development runway that feeds into that production path.

## Getting Started

To explore Container Apps Express today:

1. Visit the [Express overview documentation](https://learn.microsoft.com/en-us/azure/container-apps/express-overview)
2. Try the streamlined portal experience at [containerapps.azure.com](https://containerapps.azure.com/)
3. Review [Container Apps scaling documentation](https://learn.microsoft.com/en-us/azure/container-apps/scale-app) for when you need advanced scaling rules
4. Explore the [Container Apps ARM/Bicep API specifications](https://learn.microsoft.com/en-us/azure/container-apps/azure-resource-manager-api-spec) for Infrastructure as Code patterns

Express is a preview feature - experiment with it for development and internal tools today, and plan your production migration path to standard Container Apps as Express matures and gains the identity, networking, and compliance features that government workloads demand.
