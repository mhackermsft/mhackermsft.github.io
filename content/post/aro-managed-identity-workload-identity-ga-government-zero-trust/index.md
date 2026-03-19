---
title: 'No More Secrets in the Cluster: ARO Managed Identity and Workload Identity Are Now GA'
date: 2026-03-19T13:45:17+00:00
author: Mike Hacker
tags:
- Security
- App Modernization
- Announcements
- Zero Trust
categories:
- App Modernization
summary: Azure Red Hat OpenShift now supports Managed Identity and Workload Identity as generally available features, eliminating long-lived credentials from containerized government workloads and advancing Zero Trust architecture.
draft: false
image_prompt: A dramatic close-up of nine ornate golden keys arranged in a radial pattern around a central glowing shield, hovering above a dark void. Each key emits a soft blue bioluminescent light from its bow, suggesting digital energy. The central shield pulses with an azure glow, and fine chain links connect each key to the shield like the spokes of a wheel. The background is deep black with subtle geometric hexagonal patterns barely visible in dark charcoal tones. Volumetric light rays emanate from the shield upward. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
---

## The Credential Problem Hidden Inside Your Containers

Every Kubernetes cluster has a secret problem - sometimes literally. Containerized applications routinely need to authenticate to cloud services: pulling images from a registry, reading configuration from Key Vault, writing data to storage, calling APIs. For years, the dominant answer to "how does your pod authenticate?" was a service principal backed by a client secret: a long-lived password stored in a Kubernetes secret, rotated manually (or not at all), and carrying full permissions until someone remembered to revoke it.

For government IT organizations that have made Zero Trust commitments - whether under pressure from OMB M-22-09, CISA's Zero Trust Maturity Model v2.0, or internal security policy - this pattern is a direct contradiction. Long-lived credentials are exactly what Zero Trust architecture is designed to eliminate. They violate the principle of least privilege, they accumulate over time, and when they are compromised, the blast radius can be severe.

The general availability of Managed Identity and Workload Identity support in Azure Red Hat OpenShift (ARO) closes this gap for organizations running containerized workloads on OpenShift. It replaces the old credential model with short-lived, automatically rotated tokens backed by Microsoft Entra ID - no secrets to store, no passwords to rotate, no credentials that expire years from now.

---

## What Changed: Managed Identity and Workload Identity Now GA in ARO

Azure Red Hat OpenShift is Microsoft and Red Hat's jointly engineered, fully managed OpenShift service on Azure. It handles patching, upgrades, and operational management of control plane and worker nodes, allowing government teams to focus on applications rather than infrastructure. Historically, ARO clusters used a service principal - an Entra ID application registration with a client secret - for the cluster itself to authenticate to Azure services. That service principal required careful management: secrets had to be rotated before expiration, and permissions had to be tracked manually.

With the GA release of Managed Identity support for ARO, that model is replaced. New ARO clusters can now be created with a full managed identity architecture that covers both the cluster infrastructure itself and the application workloads running within it. This is a two-layer capability:

### Layer 1 - Cluster-Level Managed Identities

ARO's core operators - the automated controllers that manage cluster infrastructure - are each assigned their own dedicated user-assigned managed identity in Microsoft Entra ID. These include:

- **Image Registry Operator** - manages container image storage
- **Network Operator** - configures cloud networking components
- **Disk Storage Operator** - manages persistent disk volumes
- **File Storage Operator** - manages file-based persistent storage
- **Cluster Ingress Operator** - manages external traffic routing
- **Cloud Controller Manager** - integrates with Azure compute and load balancers
- **Machine API Operator** - manages node provisioning and scaling
- **ARO Service Operator** - manages the ARO service layer itself

An additional ninth identity, the **Cluster Identity (aro-cluster)**, acts as the orchestrating identity that enables federated credential creation for all the above operators. Each operator identity is granted only the specific Azure built-in roles required for its function, following the principle of least privilege at the infrastructure level. For detailed role assignments, see the [Azure Red Hat OpenShift managed identities documentation](https://learn.microsoft.com/en-us/azure/openshift/howto-understand-managed-identities).

### Layer 2 - Workload Identity for Applications

For the applications your developers deploy inside the cluster, ARO supports Microsoft Entra Workload ID. This uses OpenID Connect (OIDC) federation - the ARO cluster acts as an OIDC token issuer, and Kubernetes service account tokens are exchanged for short-lived Microsoft Entra access tokens. Applications use this mechanism to access Azure services like Key Vault, Storage, or SQL without storing any credentials.

The process works as follows:
1. A Kubernetes service account is annotated with the client ID of a user-assigned managed identity
2. A mutating webhook injects environment variables into pods that enable the OIDC token exchange
3. The application uses the Azure Identity SDK's `DefaultAzureCredential` or `WorkloadIdentityCredential`, which automatically handles the token exchange
4. Microsoft Entra ID validates the Kubernetes service account token and issues a short-lived access token scoped to the exact resources the application needs

This means a pod reading secrets from Azure Key Vault never stores a password anywhere. It presents a cryptographically signed Kubernetes token, exchanges it for a time-limited Entra token, and accesses only the resources it is explicitly authorized to access. When the token expires - typically within an hour - it is automatically refreshed. There is no long-lived credential to steal.

For implementation guidance across .NET, Go, Java, Python, and Node.js, see [Workload Identity in AKS](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview) (the same Azure Identity SDK libraries work across both AKS and ARO workloads).

---

## Why This Matters for Government

Government IT organizations face a uniquely demanding credential security environment. Public records laws, audit requirements, and the sensitivity of citizen data all raise the stakes when credentials are mismanaged. Several federal and state-level mandates make the move away from long-lived credentials not just a best practice but an emerging compliance requirement:

**CISA Zero Trust Maturity Model v2.0** identifies identity as one of its five core pillars, and explicitly calls for moving toward phishing-resistant, continuously verified authentication for all workloads - not just human users. Non-human identities (service accounts, application credentials, API keys) are increasingly in scope for Zero Trust programs. The [CISA ZTMM](https://www.cisa.gov/zero-trust-maturity-model) provides a maturity progression from "Traditional" to "Optimal" - and eliminating long-lived credentials from containerized workloads is a clear step toward "Advanced" or "Optimal" posture.

**OMB M-22-09** (Federal Zero Trust Strategy) specifically calls for agencies to treat identity as the new perimeter and to move toward phishing-resistant MFA and short-lived credentials. While OMB directives apply directly to federal agencies, many state and local governments use them as authoritative guidance for their own modernization roadmaps.

**Audit and Forensics**: Managed identity authentication generates sign-in activity in Microsoft Entra ID sign-in logs. Each token exchange is recorded with the identity, resource accessed, timestamp, and result. For government organizations that must demonstrate who accessed what and when, this provides a clean, centralized audit trail that service principal client secrets cannot match.

**Credential Sprawl Elimination**: Government organizations often operate many containerized applications across multiple teams and environments. Service principal credentials accumulate over time - some tied to departed employees, some never rotated, some with overly broad permissions granted during a sprint and never revisited. Managed identity eliminates this accumulation. When an application or cluster is decommissioned, the managed identity is deleted and access is automatically revoked with no orphaned credentials remaining.

**Supply Chain Security**: A compromised container image or dependency package that exfiltrates environment variables cannot steal a long-lived credential if no long-lived credential exists. Workload Identity tokens are short-lived and bound to the specific Kubernetes service account and pod, dramatically reducing the value of any credential theft attempt.

---

## Important Deployment Considerations

**New clusters only**: Existing ARO clusters using service principals cannot be migrated in-place to managed identity. Organizations planning to adopt this capability will need to provision new clusters. This is worth factoring into modernization roadmaps now - particularly for organizations planning ARO deployments or major cluster upgrades in 2025 or 2026.

**Azure CLI version requirement**: Creating a managed identity ARO cluster requires Azure CLI version 2.84.0 or higher. Verify your tooling version before beginning.

**Nine managed identities to pre-create**: The cluster creation process requires pre-creating eight operator managed identities and one cluster identity, each with specific role assignments scoped to the appropriate resource groups. The [ARO cluster creation guide](https://learn.microsoft.com/en-us/azure/openshift/howto-create-openshift-cluster) walks through this process in detail. Consider using infrastructure-as-code (Bicep or ARM templates) to make this repeatable across environments.

**Azure Commercial vs. Azure Government**: ARO is available in Azure commercial regions today. Government organizations running workloads in Azure Government (USGov Virginia, USGov Arizona) should verify current regional availability for ARO through the [Azure products by region](https://azure.microsoft.com/en-us/global-infrastructure/services/?products=openshift) page and consult their Microsoft account team for the most current guidance on Azure Government availability timelines.

**M365 GCC tenants**: Organizations operating in M365 GCC tenants that also use Azure commercial for containerized workloads can leverage ARO's managed identity capabilities in the commercial Azure environment. Identity federation through Microsoft Entra ID works across these boundaries - workload identities in ARO commercial clusters can be configured to access resources authorized through the same Entra tenant backing your GCC environment.

---

## Getting Started

For government development teams ready to adopt this capability, here is the recommended path:

1. **Understand the identity model**: Review [Understanding managed identities in Azure Red Hat OpenShift](https://learn.microsoft.com/en-us/azure/openshift/howto-understand-managed-identities) to understand the nine-identity architecture and role assignment model before provisioning.

2. **Plan your resource groups**: The managed identity role assignments are scoped to specific resource groups. Align this with your existing resource group design and network topology before creating identities.

3. **Create the cluster with managed identity**: Follow the [ARO managed identity cluster creation guide](https://learn.microsoft.com/en-us/azure/openshift/howto-create-openshift-cluster) using Azure CLI, Bicep, or ARM templates. Build this as repeatable IaC from the start.

4. **Adopt Workload Identity in your applications**: Update application code to use `DefaultAzureCredential` from the Azure Identity SDK. This single credential type automatically supports both managed identity and workload identity contexts, making applications portable across local development (using developer credentials) and production (using workload identity).

5. **Audit your existing service principal inventory**: Use Microsoft Entra's access reviews for service principals to identify which existing credentials can be retired as you migrate workloads to managed identity-capable clusters.

---

## The Bigger Picture: Identity as Infrastructure

For government IT leaders, the GA of Managed Identity and Workload Identity in ARO is not just a developer convenience feature - it is a material change in the security posture of containerized workloads. It makes the elimination of long-lived credentials achievable without requiring application code changes in most cases, and without creating new operational burden around secret rotation.

As government organizations modernize legacy applications, adopt microservices patterns, and expand container platforms, the identity model underlying those containers becomes increasingly important. Building on managed identity from the start means that identity is treated as infrastructure - provisioned, governed, and audited through the same processes as compute and networking - rather than as an afterthought left to individual developers to manage.

Zero Trust is not a product you buy or a checklist you complete. It is an ongoing architectural commitment to reducing implicit trust at every layer. Eliminating long-lived credentials from containerized workloads is one of the most impactful steps that government IT organizations can take toward that goal - and Azure Red Hat OpenShift's GA support for Managed Identity and Workload Identity makes it achievable today.

---

## Learn More

- [Understanding managed identities in Azure Red Hat OpenShift](https://learn.microsoft.com/en-us/azure/openshift/howto-understand-managed-identities)
- [Create an ARO cluster with managed identities](https://learn.microsoft.com/en-us/azure/openshift/howto-create-openshift-cluster)
- [What are managed identities for Azure resources?](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)
- [Microsoft Entra Workload ID overview](https://learn.microsoft.com/en-us/entra/workload-id/workload-identities-overview)
- [Workload Identity in Azure Kubernetes Service](https://learn.microsoft.com/en-us/azure/aks/workload-identity-overview)
- [CISA Zero Trust Maturity Model v2.0](https://www.cisa.gov/zero-trust-maturity-model)
- [Azure Red Hat OpenShift product page](https://azure.microsoft.com/en-us/products/openshift/)

