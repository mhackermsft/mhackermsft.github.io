---
title: 'Claude on Microsoft Foundry vs Anthropic Direct: A Governance and Architecture Decision Guide'
date: 2026-06-30T12:56:01+00:00
author: Mike Hacker
tags:
- AI
- Security
categories:
- Azure AI
- Government
summary: Anthropic Claude models are available through Microsoft Foundry with Hosted on Azure and Hosted on Anthropic infrastructure options. This guide compares Foundry and Anthropic direct paths for data handling, identity, RBAC, networking, compliance, and billing.
draft: false
image_prompt: A clean government technology architecture illustration showing Microsoft Foundry as a central Azure governance hub connected to Claude model options, Microsoft Entra identity, Azure billing, private networking, compliance review, and an external Anthropic API path. Professional public-sector tone, Azure blue palette, no logos beyond abstract cloud and security icons.
image: cover.png
audio: audio.mp3
---

Anthropic's Claude family is available in the Microsoft Foundry model catalog, and that changes the procurement and architecture conversation for public sector technology teams. The question is not only "which model do we use," but "which control plane, identity system, billing path, and data-processing terms govern the workload."

Consuming Claude directly from Anthropic and consuming Claude through Microsoft Foundry can involve the same model family, but the operational worlds are different. Authentication, networking, billing, support boundaries, and data-processing terms all need to be reviewed before an agency standardizes on a path.

This guide summarizes the decision points using current Microsoft and Anthropic documentation.

## Two Foundry categories, and two Claude hosting paths

Microsoft Foundry organizes its model catalog into two main categories: **Foundry Models sold by Azure** and **Foundry Models from partners and community**. The distinction matters.

**Models sold by Azure**, also called Azure Direct models or Direct from Azure models, are hosted and sold by Microsoft under Microsoft Product Terms. Microsoft documentation says these models are billed through an Azure subscription, covered by Azure service-level agreements, supported by Microsoft, and reviewed under Microsoft's Responsible AI standards.

**Models from partners and community** are different. Microsoft documentation states that partner and community models that are not sold by Azure are Non-Microsoft Products under the Product Terms. The provider defines the license terms and pricing through Azure Marketplace, and provider terms can govern data processing.

Claude is documented in the partner and community category. The current Microsoft documentation says Claude models in Foundry can appear in two versions: **Hosted on Azure** and **Hosted on Anthropic infrastructure**. Both versions are not available for every model, and the lifecycle stage, such as Preview or Generally Available, can differ between the two versions.

That gives government teams three practical consumption paths:

1. **Anthropic direct**: Use Anthropic's Claude API at `https://api.anthropic.com` with Anthropic authentication, billing, terms, and support.
2. **Foundry, Hosted on Anthropic infrastructure**: Provision through Microsoft Foundry and Azure Marketplace while inference is hosted on Anthropic infrastructure.
3. **Foundry, Hosted on Azure**: Use the Claude option identified by Microsoft as hosted on Azure, while still treating Claude as a partner model and validating the exact model card, terms, lifecycle stage, region, and data flow.

## Data residency and data handling

For a state or local agency, data handling is usually the first review item. The key nuance is that a model's presence in the Foundry catalog does not automatically mean Microsoft Product Terms govern all aspects of the model. For partner and community models that are not sold by Azure, Microsoft documentation points customers to the provider's terms. For Claude, the Foundry documentation links to Anthropic's Data Processing Addendum and Commercial Terms of Service.

Practically, the three paths differ:

**Anthropic direct** sends requests to Anthropic's Claude API and uses Anthropic's commercial relationship, authentication model, support path, data terms, and regional availability policy.

**Foundry, Hosted on Anthropic infrastructure** gives teams a Microsoft Foundry provisioning and Azure Marketplace procurement path, but the model is still hosted on Anthropic infrastructure. Data-flow and residency review should follow the specific Claude model documentation and Anthropic terms.

**Foundry, Hosted on Azure** is the option agencies will typically examine first when they want the deployment to align more closely with Azure controls. Even then, teams should not assume that "Hosted on Azure" means every compliance requirement is satisfied. Confirm the specific model version, deployment option, lifecycle stage, supported regions, and data-processing terms before committing regulated data.

The safe governance rule is simple: do not treat catalog presence as a residency guarantee. Treat the model card, deployment details, provider terms, and region support as the source of truth.

## Identity and RBAC

This is one of the clearest operational differences.

Anthropic direct access uses Anthropic's own API authentication model. Anthropic documentation states that Claude API requests require either an API key or an Authorization bearer token obtained through Workload Identity Federation, plus the required `anthropic-version` and `content-type` headers. That can be appropriate for teams that already manage Anthropic as a separate platform, but it is still a separate identity and access-management surface from Azure RBAC and Microsoft Entra governance.

Foundry keeps access closer to the Azure control plane. Microsoft documentation for model deployment options states that managed compute and serverless deployments support keys and Microsoft Entra authentication. For organizations that already use managed identities, Azure RBAC, conditional access, Privileged Identity Management, and centralized logging, this can reduce the number of standalone provider secrets and external access paths that security teams must govern.

Do not copy an endpoint pattern from a blog post into production. Use the endpoint, API path, request schema, token scope, and role assignments shown for your exact deployment in the Foundry portal and current Microsoft documentation. The current Azure AI Model Inference REST reference documents bearer-token authorization and `POST /chat/completions?api-version=2025-04-01` for the common inference API, while Claude-specific deployments can have model-specific requirements. Validate the generated endpoint and schema for the specific Claude version you deploy.

There is also a Marketplace permission requirement. Microsoft documentation lists the Azure actions required to subscribe to partner and community models. A custom role can include these action strings, along with your normal role metadata and assignable scopes:

- `Microsoft.MarketplaceOrdering/agreements/offers/plans/read`
- `Microsoft.MarketplaceOrdering/agreements/offers/plans/sign/action`
- `Microsoft.MarketplaceOrdering/offerTypes/publishers/offers/plans/agreements/read`
- `Microsoft.Marketplace/offerTypes/publishers/offers/plans/agreements/read`
- `Microsoft.SaaS/register/action`
- `Microsoft.SaaS/resources/read`
- `Microsoft.SaaS/resources/write`

The Microsoft documentation also states that the built-in Owner and Contributor roles on the Azure subscription include these permissions, and that `Microsoft.SaaS/register/action` is a one-time registration of the SaaS resource provider on the subscription.

## Networking and private connectivity

Direct Anthropic usage means your application reaches Anthropic's API endpoint outside Azure. That may be acceptable for some workloads, but it typically requires explicit egress review, proxy policy, monitoring, and data-loss-prevention controls.

Foundry gives agencies an Azure-native network-control surface. Microsoft documentation for Foundry network isolation describes private endpoints, the public network access flag, private DNS, and patterns for controlling inbound access to Foundry resources and projects. The Foundry Models overview also describes network isolation behavior for managed compute and serverless deployment options, including managed networks and public network access settings.

The design implication is important: with Foundry, Claude access can be reviewed as part of the same Azure networking pattern used for other AI, data, and application services. With Anthropic direct, the agency must separately govern outbound connectivity to Anthropic.

## Compliance, content safety, and governance

Foundry does not eliminate the need for model due diligence, but it can simplify platform governance.

Microsoft documentation says Azure AI Content Safety filtering is integrated with serverless inference APIs for Foundry model deployments, and that these filters are billed separately. The Content Safety pricing page confirms that Content Safety has its own billing model. That gives agencies a way to apply a more consistent safety-control layer across model choices where the deployment type supports it.

The support and terms boundary still matters. Microsoft documentation says to use Microsoft Support for help with Anthropic models in Foundry, while the model provider and provider terms remain relevant for Claude behavior, privacy, and data processing. For Models sold by Azure, Microsoft documents Azure billing, Microsoft support, and Azure service-level agreements. For partner models, read the model card and provider terms.

For regulated workloads, do not collapse platform compliance into model approval. Azure and Azure Government have compliance documentation for programs such as FedRAMP and CJIS, but an agency still needs to confirm whether the specific service, cloud, region, model, deployment type, and data path are in scope for its authorization package.

## Billing, procurement, and subscription blockers

Anthropic direct means a direct Anthropic commercial relationship and Anthropic billing. Anthropic's Commercial Terms state that customers are responsible for fees incurred by their account at the rates specified by Anthropic unless otherwise agreed.

Foundry partner models are purchased through Azure Marketplace. Microsoft documentation says providers define the license terms and price for partner and community models using Azure Marketplace. For many government finance teams, that can be easier to align with existing Azure billing processes than creating a separate vendor path.

Be precise about cost reporting. Microsoft Cost Management documentation says Cost Management includes Marketplace usage and purchases for Enterprise Agreement and Microsoft Customer Agreement accounts, with offer-specific caveats. It also documents how tags appear in cost and usage data, including limitations such as tag support by resource type and timing. In other words, Foundry can improve cost visibility, but tagging and cost allocation should be tested in the billing account type your agency actually uses.

Two subscription constraints are critical for Claude in Foundry. Microsoft documentation says Claude requires a paid Azure subscription with a billing account in a country or region where Anthropic offers the model for purchase. The documentation also lists unsupported subscription types, including Cloud Solution Provider subscriptions, Enterprise accounts located in South Korea, Azure subscriptions without an active pay-as-you-go billing method, and sponsored subscriptions that only use Azure credits.

Many public sector organizations buy through partner-led procurement or operate in constrained cloud environments. Validate subscription type, billing account, region, cloud, and model availability before designing around Claude in Foundry.

## Why This Matters for Government

State and local agencies rarely adopt AI capability in isolation. The capability arrives inside a control plane, and that control plane is what auditors, finance officers, security architects, procurement teams, and privacy officers must live with.

For workloads tied to CJIS, IRS Pub 1075, HIPAA, StateRAMP-style evidence reviews, FedRAMP-aligned authorization packages, or local data-residency commitments, the hosting and control-plane decision can matter as much as the model choice. Foundry can help agencies keep more of the lifecycle inside Azure governance: Microsoft Entra authentication, Azure RBAC, private networking patterns, Azure Marketplace procurement, Cost Management, and consistent content-safety controls. That is valuable when teams need repeatable architecture patterns rather than one-off AI integrations.

Anthropic direct can still make sense. It may be the right path for a prototype, a workload that depends on an Anthropic feature not yet available in Foundry, or an organization that already has an Anthropic enterprise agreement and governance process. But it creates a separate platform surface for identity, network egress, billing, monitoring, and terms review.

For most public sector production planning, start with Foundry and evaluate the two Claude hosting versions. Prefer the Azure-hosted option when the model version, lifecycle stage, region, subscription eligibility, and data-processing review satisfy the workload. Use the Anthropic-hosted Foundry path when procurement through Azure matters but the required model is not available in an Azure-hosted version. Use Anthropic direct when feature availability or existing contractual structure outweighs the benefits of Azure control-plane integration.

The right move is to treat the Claude hosting decision as a governance decision. Verify the current model card, deployment type, subscription eligibility, region support, data terms, identity model, and network path before regulated data enters the system.

## References

- [Microsoft Foundry Models overview](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/foundry-models-overview)
- [Foundry Models sold by Azure](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-models/concepts/models-sold-directly-by-azure)
- [Foundry Models from partners and community](https://learn.microsoft.com/en-us/azure/ai-foundry/foundry-models/concepts/models-from-partners)
- [Azure AI Model Inference REST API reference](https://learn.microsoft.com/en-us/azure/ai-foundry/model-inference/reference/reference-model-inference-api)
- [Configure network isolation for Microsoft Foundry](https://learn.microsoft.com/en-us/azure/ai-foundry/how-to/configure-private-link)
- [Content filtering for Microsoft Foundry Models](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/content-filtering)
- [Content Safety in Foundry Control Plane pricing](https://azure.microsoft.com/en-us/pricing/details/cognitive-services/content-safety/)
- [Understand Cost Management data](https://learn.microsoft.com/en-us/azure/cost-management-billing/costs/understand-cost-mgt-data)
- [Use tags to organize Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources)
- [Anthropic Claude API overview](https://docs.anthropic.com/en/api/overview)
- [Anthropic Commercial Terms of Service](https://www.anthropic.com/legal/commercial-terms)
- [Anthropic Data Processing Addendum](https://www.anthropic.com/legal/data-processing-addendum)
- [Azure Government and global Azure comparison](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure)
- [Azure FedRAMP compliance documentation](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-fedramp)
- [Azure CJIS compliance documentation](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-cjis)
