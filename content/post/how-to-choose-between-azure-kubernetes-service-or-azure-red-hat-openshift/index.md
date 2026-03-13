---
title: 'How to choose between Azure Kubernetes Service or Azure Red Hat OpenShift'
date: 2023-04-21T18:25:17.794Z
lastmod: 2023-04-21T18:41:49.914Z
author: Mike Hacker
url: /how-to-choose-between-azure-kubernetes-service-or-azure-red-hat-openshift/
tags:
  - 'Articles'
summary: "Note: Portions of this blog post were written using Bing Chat. Azure Kubernetes Service (AKS) and Azure Red Hat OpenShift (ARO) are two options for deploying..."
draft: false
image: cover.jpg
---

> *Note: Portions of this blog post were written using Bing Chat.*

Azure Kubernetes Service (AKS) and Azure Red Hat OpenShift (ARO) are two options for deploying and managing containerized applications on Azure. Both services are based on Kubernetes, the open-source platform for orchestrating containers, but they have some differences and trade-offs that you should consider before choosing one over the other.

AKS is a fully managed Kubernetes service that offers serverless Kubernetes, an integrated continuous integration and continuous delivery (CI/CD) experience, and enterprise-grade security and governance. AKS is optimized for running general purpose containers, especially for applications that span many microservices deployed in containers. AKS supports Kubernetes-style apps and microservices with features like service discovery and traffic splitting. AKS also enables event-driven application architectures by supporting scale based on traffic and pulling from event sources like queues, including scale to zero. AKS does not provide direct access to the underlying Kubernetes control plane, which means you have less control and customization options over your cluster.

ARO is a fully managed OpenShift service that provides a consistent hybrid cloud foundation for building and scaling containerized applications. OpenShift is a distribution of Kubernetes that adds additional features and integrations to enhance the developer experience and enterprise readiness of Kubernetes. OpenShift provides a web console, an integrated image registry, a built-in CI/CD pipeline, a source-to-image tool, a service catalog, and many other tools to help developers build, deploy, and manage applications. OpenShift also offers more security features than AKS, such as role-based access control, security context constraints, network policies, and encryption. ARO provides direct access to the underlying Kubernetes APIs and control plane, which gives you more flexibility and customization options over your cluster.

The pros and cons of each service depend on your specific use case and requirements. Some general considerations are:

\- If you want a fully managed Kubernetes experience with minimal configuration and maintenance, AKS may be a good choice.

\- If you want a more comprehensive platform that includes additional tools and integrations for developing and deploying containerized applications, ARO may be a better option.

\- If you need direct access to the Kubernetes control plane, or if you want to use OpenShift-specific features or extensions, ARO is the only option.

\- If you need to scale your applications based on events or traffic, or if you want to leverage serverless containers, AKS has more support for these scenarios than ARO.

\- If you are concerned about security and compliance, ARO may offer more features and capabilities than AKS to meet your needs.

Ultimately, the best way to choose between AKS and ARO is to try them out for yourself and see which one suits your needs better. You can get started with AKS using the quickstarts or with ARO using the tutorials.
