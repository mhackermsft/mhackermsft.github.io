---
title: 'Event: Azure Lab Services Retirement - Free Webinar on Planning Your Transition'
date: 2026-03-24T12:39:24+00:00
author: Mike Hacker
tags:
- Events
- App Modernization
- Announcements
categories:
- Events
summary: Join a free webinar on March 26 from Nerdio and Microsoft covering the Azure Lab Services retirement timeline and how to plan a smooth transition to modern alternatives.
draft: false
image_prompt: A dramatic scene of an old ornate iron bridge being carefully dismantled on one side while a sleek modern suspension bridge of glass and steel rises beside it, both spanning a deep misty canyon. Golden sunset light pours through the cables of the new bridge. Construction cranes lift gleaming metallic sections into place. Scattered gears and bolts from the old bridge rest on the canyon rim in the foreground. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
---

Azure Lab Services will be **fully retired on June 28, 2027**. If your organization relies on Azure Lab Services for training environments, testing platforms, hackathons, or proof-of-concept deployments, the clock is ticking. While over a year remains, the best time to start planning your transition is now.

On **March 26, 2026 at 10:00 AM ET**, Nerdio and Microsoft are hosting a free webinar that walks through the retirement timeline, explains why early planning matters, and explores the alternative approaches available for running labs and training environments in Azure.

**[Register for the free webinar here](https://getnerdio.com/events/azure-lab-services-retirement-how-to-prepare-and-simplify-your-transition/)**

## What Is Azure Lab Services and Why Is It Retiring?

Azure Lab Services has been a managed platform that allowed organizations to quickly provision virtual machine-based lab environments for training, development, testing, and classroom scenarios. It abstracted away much of the infrastructure complexity, making it straightforward for IT administrators to spin up VMs pre-loaded with specific software configurations.

Microsoft [announced the retirement](https://learn.microsoft.com/en-us/azure/lab-services/retirement-guide) as part of a broader strategy to consolidate virtualization workloads around more capable, enterprise-grade services. As of **July 15, 2024**, new customers can no longer sign up for Azure Lab Services. Existing lab accounts, lab plans, and labs will continue to operate until the **June 28, 2027** retirement date, after which the service will no longer be supported and you will lose access to lab accounts, lab plans, and labs. Importantly, any images saved to an Azure Compute Gallery will remain accessible.

Microsoft continues to provide full support for existing Azure Lab Services deployments until the retirement date, including fixes for blocking bugs and security issues. However, Microsoft strongly recommends beginning your transition planning now to address pricing differences, secure necessary compute cores, and take advantage of enhanced features available in replacement solutions.

## What the Webinar Covers

This practical session features speakers from both Nerdio and Microsoft:

- **Dave Fiske** - Lead EUC Strategist, Nerdio
- **Sean Corrigan** - Sales Engineer, Nerdio
- **Mike Smucny** - Cloud and AI Solution Engineering Leader, SLED, Microsoft

The session will address four key areas:

1. **Key dates in the Azure Lab Services retirement timeline** and what they mean for your existing environments
2. **Why organizations should begin planning now** instead of waiting until closer to the deadline
3. **Alternative approaches** for running labs and training environments in Azure
4. **How Nerdio can simplify the transition** from Azure Lab Services

The webinar closes with a **live Q&A** so you can ask your questions directly to the experts.

## Understanding Your Transition Options

Microsoft has outlined several first-party and partner solutions as recommended replacements. Understanding the landscape of alternatives is critical for selecting the right fit for your organization.

### Microsoft First-Party Solutions

**[Azure Virtual Desktop (AVD)](https://learn.microsoft.com/en-us/azure/virtual-desktop/overview)** is a comprehensive desktop and app virtualization service running in the cloud. It supports full desktop delivery, RemoteApp for individual applications, multi-session Windows 10/11 capabilities, and browser-based access. AVD offers usage-based pricing, autoscale capabilities, and integration with Microsoft Entra ID and Intune. For organizations that need maximum control and flexibility, AVD is often the recommended path forward.

**[Azure DevTest Labs](https://learn.microsoft.com/en-us/azure/devtest-labs/devtest-lab-overview)** simplifies the creation and management of IaaS virtual machines within a lab context. It is particularly well-suited for computer programming-related scenarios, Linux VM requirements, and environments that leverage nested virtualization. DevTest Labs is free to use as a service; you only pay for the underlying Azure resources consumed. Microsoft has published a detailed [transition guide from Azure Lab Services to DevTest Labs](https://learn.microsoft.com/en-us/azure/lab-services/transition-devtest-labs-guidance).

**[Windows 365 Cloud PC](https://www.microsoft.com/windows-365)** provides a persistent, scalable virtual machine with predictable subscription pricing. Cloud PCs are Microsoft Entra ID joined and managed through Microsoft Intune, making them an excellent option for organizations already invested in the Microsoft endpoint management ecosystem.

**[Microsoft Dev Box](https://learn.microsoft.com/en-us/azure/dev-box/overview-what-is-microsoft-dev-box)** offers cloud-based developer workstations preconfigured with tools and environments tailored for development workflows. This is ideal for hands-on learning and training where all participants need identical, ready-to-use environments.

### Azure Partner Solutions

Microsoft has also endorsed several Azure partner solutions that offer specialized, education-focused capabilities:

- **[Nerdio Manager for Enterprise (NME)](https://aka.ms/azlabs-nerdio)** builds on top of Azure Virtual Desktop with added automation, cost optimization, and simplified management. Microsoft specifically recommends NME for customers transitioning Windows lab scenarios to AVD.
- **[Apporto](https://aka.ms/azlabs-apporto)**, **[CloudLabs by Spektra Systems](https://aka.ms/azlabs-spektra)**, and **[Skillable](https://aka.ms/azlabs-skillable)** each specialize in education-focused solutions supporting features like LMS integration, over-the-shoulder monitoring, and nested virtualization.

All partner solutions support browser-based web access, cost controls, Azure Compute Gallery images, and CPU/GPU-based virtual machines. Labs created with these solutions run on Azure infrastructure, where provisioning can be configured to specific Azure regions for low latency and data residency requirements.

## Why Start Planning Now?

The June 2027 deadline may feel distant, but there are compelling reasons to begin your transition strategy today:

- **Pricing model differences**: Azure Lab Services includes storage costs as a complimentary service. Replacement solutions have different pricing structures that may require budget adjustments. Understanding these differences early allows for proper fiscal planning.
- **Compute core allocation**: You may need to secure additional compute quota in your Azure subscription to support your chosen replacement. Core limit increases can take time to process.
- **Feature evaluation**: While replacement solutions generally offer more features than Lab Services (browser-based access, dynamic VM creation, multi-session capabilities), some capabilities require a different approach. Testing and validating these differences takes time.
- **Cleanup opportunity**: Microsoft has published [automation scripts on GitHub](https://github.com/microsoft/Azure-Lab-Services-Retirement-Scripts) to help organizations clean up unused Lab Services resources, reduce costs, and formally offboard. Starting this process early reduces your security surface area and eliminates waste from idle VMs.

Microsoft recommends using the latest Azure Lab Services version (utilizing lab plans rather than legacy lab accounts) as an interim solution while developing your long-term transition plan. Lab plans offer enhanced performance, a wider range of VM sizes, and faster VM creation times.

## Why This Matters for Government

Government organizations frequently use lab environments for a variety of critical functions:

- **Employee training and certification**: IT departments use lab environments to train staff on new systems, run cybersecurity awareness exercises, and prepare teams for technology migrations.
- **Testing and quality assurance**: Before deploying new applications or updates to production, government agencies need isolated environments to validate functionality and security.
- **Hackathons and innovation**: Many government technology teams run internal hackathons and innovation sprints that require quickly provisioned, temporary compute environments.
- **Proof-of-concept deployments**: Evaluating new solutions before committing procurement dollars often requires spinning up controlled lab environments.

The retirement of Azure Lab Services directly impacts these workflows. Government IT leaders need to evaluate replacement options through the lens of **compliance requirements**, **budget cycles**, and **procurement timelines** that are typically longer and more rigid than in the private sector.

Azure Virtual Desktop, one of the primary recommended alternatives, is available in both Azure commercial and Azure Government regions, which is important for agencies with data residency or compliance requirements such as FedRAMP, CJIS, or IRS 1075. Organizations operating in Azure Government should verify feature parity for their specific use case, as some capabilities may roll out on different timelines between commercial and government clouds.

For agencies that use Microsoft 365 GCC tenants, it is worth noting that Windows 365 Cloud PC and Microsoft Intune management capabilities integrate with GCC environments, though specific feature availability should be validated against Microsoft's [GCC documentation](https://learn.microsoft.com/en-us/office365/servicedescriptions/office-365-platform-service-description/office-365-us-government/gcc).

Perhaps most importantly, **government budget cycles often run 12 to 18 months ahead**. If your agency needs to allocate funding for a new lab management platform, training, or migration services, the procurement and budget request process needs to start soon to have resources in place well before the June 2027 cutoff.

## Key Takeaways

- **Azure Lab Services retires June 28, 2027**: After this date, all lab accounts, lab plans, and labs will become inaccessible.
- **New sign-ups have been closed since July 2024**: Only existing customers can continue to create lab plans.
- **Multiple transition paths exist**: Azure Virtual Desktop, DevTest Labs, Windows 365, Microsoft Dev Box, and partner solutions like Nerdio all offer replacement capabilities.
- **Start planning now**: Budget implications, compute quota needs, and government procurement timelines make early planning essential.
- **Attend the webinar for expert guidance**: The March 26 session brings together Microsoft and Nerdio experts who can answer your specific questions.

## Register for the Webinar

**Date**: March 26, 2026
**Time**: 10:00 AM ET
**Cost**: Free
**Format**: Virtual with live Q&A

**[Register now](https://getnerdio.com/events/azure-lab-services-retirement-how-to-prepare-and-simplify-your-transition/)**

## Additional Resources

- [Azure Lab Services Retirement Guide - Microsoft Learn](https://learn.microsoft.com/en-us/azure/lab-services/retirement-guide)
- [Azure Lab Services to Azure DevTest Labs Transition Guide](https://learn.microsoft.com/en-us/azure/lab-services/transition-devtest-labs-guidance)
- [Azure Lab Services Retirement Scripts - GitHub](https://github.com/microsoft/Azure-Lab-Services-Retirement-Scripts)
- [What is Azure Virtual Desktop? - Microsoft Learn](https://learn.microsoft.com/en-us/azure/virtual-desktop/overview)
- [What is Azure DevTest Labs? - Microsoft Learn](https://learn.microsoft.com/en-us/azure/devtest-labs/devtest-lab-overview)
- [What is Microsoft Dev Box? - Microsoft Learn](https://learn.microsoft.com/en-us/azure/dev-box/overview-what-is-microsoft-dev-box)

