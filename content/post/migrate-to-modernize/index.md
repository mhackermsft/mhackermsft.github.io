---
title: 'Migrate to Modernize'
date: 2023-05-24T15:13:37.343Z
lastmod: 2023-09-26T13:56:52.086Z
author: Mike Hacker
url: /migrate-to-modernize/
tags:
  - 'Articles'
summary: 'Migrate to Modernize is an approach for application modernization that enables organizations to see immediate value while also addressing common modernizatio...'
draft: false
image: cover.png
---

Migrate to Modernize is an approach for application modernization that enables organizations to see immediate value while also addressing common modernization challenges.

Modernizing an application means to pay back that technical debt and to enable the application to take advantage of modern platform services. Modern platform services enables organizations to focus more on solving business challenges and provides more value by not having to focus on hardware, patching and server maintenance.

The fastest way to modernize is to first migrate your applications to Azure. Once the application is in Azure you can begin to modernize parts of the solution by replacing virtual machines with platform services. For example, you can replace the SQL database VM with Azure SQL, that IIS server with Azure App Service, and modernize the applications identity to use Azure Active Directory for authentication.

[Here is a short video](https://www.linkedin.com/posts/mphacker_msftadvocate-azure-government-activity-7056263142070243328-a9sM?utm_source=share&utm_medium=member_desktop) that explains the Migrate to Modernize approach.

Below are a few recommendations and things to consider with using the Migrate to Modernize approach:

-   Quickly get started by modernizing the database first by utilizing Azure SQL Database, Azure SQL Managed Instance, MySQL, or PostgreSQL platform services.
-   Migrate the application VMs to Azure and reconfigure the application to utilize the new hosted database service
-   Establish policies that incentivize completing the full application modernization process within an acceptable timeframe.
-   Developers should modernize the application to utilize modern authentication solutions such as Azure Active Directory and Azure AD B2C.
-   Developers should upgrade the code to utilize the latest frameworks and libraries to help protect against security vulnerabilities. 
-   Architects and developers can work together to determine the best platform services to use for hosting the updated application. Options include services such as Azure Kubernetes Service, Azure Container Apps, Azure Red Hat OpenShift, and Azure App Service.
-   Architects and developers can consider decomposing the application into microservices and APIs that can be hosted on container or serverless platforms.
-   Architects and developers can begin to look for options to enhance their application with other platform as a service features such as Logic Apps, Service Bus, Redis Cache, Configuration Services, Functions.

The key point is that you start with modernizing the database while migrating the application. From there you can begin to modernize components of the app while receiving the benefits of using database platform services and having access to a wide range of additional platform services for your application.

To get started today, speak with your Microsoft Account Team for details on how they can assist you with your modernization journey.
