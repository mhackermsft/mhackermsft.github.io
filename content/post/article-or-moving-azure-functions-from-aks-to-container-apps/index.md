---
title: "Article | Moving Azure Functions from AKS to Container Apps"
date: 2022-01-26T14:47:48.269Z
lastmod: 2022-01-26T15:05:09.939Z
author: Mike Hacker
url: /article-or-moving-azure-functions-from-aks-to-container-apps/
tags:
  - 'Articles'
summary: "Microsoft Azure Functions enables users to execute event-driven serverless code which can be hosted in a wide range of environments. Hosting options include:..."
draft: false
image: cover.webp
---

Microsoft Azure Functions enables users to execute event-driven serverless code which can be hosted in a wide range of environments. Hosting options include:

-   Consumption Plan - scale automatically and only pay for the compute resources the function is running.
-   Premium Plan - Automatically scales based on demand using pre-warmed workers which run applications with no delay after being idle, runs on more powerful instances, and connects to virtual networks.
-   Dedicated Plan - Run your Azure Functions within an App Service plan at the regular App Service plan rates.
-   App Service Environment - Fully isolated and dedicated environment for running App Service apps and Azure Functions at high scale.

Azure Functions can also be deployed using containers on Kubernetes, Azure Arc, or even the new Azure Container Apps.

Christian Leinweber recently posted an article that outlines how he did a proof of concept on moving Azure Functions from Azure Kubernetes Service to Container Apps. This is a great read and shows the power of combining Azure Functions with other Azure services. [Check out the article here](https://dev.to/christle/moving-azure-functions-from-aks-to-container-apps-k60)!
