---
title: 'Turning Clinical Notes Into Structured Data: A Technical Deep-Dive on Text Analytics for health'
date: 2026-06-04T15:41:08+00:00
author: Mike Hacker
tags:
- AI
- Data Platform
- Security
- How To
categories:
- AI
- Healthcare
- Government
summary: How government health and human services agencies can extract structured clinical insight from unstructured documents using clinical NER, relation extraction, UMLS entity linking, FHIR output, PHI de-identification, and containers.
draft: false
image_prompt: Technical illustration of a government health document-processing pipeline turning clinical notes into structured entities, FHIR bundles, privacy redaction, and secure Azure Government storage.
image: cover.png
audio: audio.mp3
---

Health and human services (HHS) agencies sit on mountains of unstructured clinical text: intake notes, discharge summaries, case worker narratives, eligibility documentation, and OCR-extracted correspondence. Azure Language in Foundry Tools includes the Text Analytics for health capability, which helps turn that free text into structured clinical data through a managed API or a self-hosted container.

This post reflects current Microsoft Learn documentation, including the Text Analytics for health overview last updated March 30, 2026, and the PII overview last updated June 2, 2026.

> **Important Microsoft disclaimer:** Text Analytics for health is provided "AS IS" and "WITH ALL FAULTS." It is not a medical device, clinical support, diagnostic tool, or substitute for professional medical judgment. Agencies must keep humans in the loop for decisions that affect individuals or resource allocation, and must separately license any UMLS Metathesaurus source vocabularies they use. See the [overview documentation](https://learn.microsoft.com/en-us/azure/ai-services/language-service/text-analytics-for-health/overview).

## What the capability actually does

Text Analytics for health performs four functions in one call:

1. **Named Entity Recognition (NER)** - extracts medical concepts such as diagnosis, medication name, dosage, frequency, symptom/sign, and age. The model can also surface Social Determinants of Health (SDOH) and ethnicity mentions, while not drawing inferences from them.
2. **Relation extraction** - identifies connections such as `DosageOfMedication`, `FrequencyOfMedication`, and `TimeOfCondition`.
3. **Entity linking** - maps extracted entities to preferred names and codes in biomedical vocabularies supported by the [UMLS Metathesaurus](https://www.nlm.nih.gov/research/umls/index.html).
4. **Assertion detection** - adds context such as certainty, conditionality, association, and temporality, which helps distinguish a denied condition from a positive finding.

Language support depends on deployment. The hosted API supports English, Spanish, French, German, Italian, and Portuguese. The Docker container supports those languages plus Hebrew when the appropriate language-family image tag is used. The service can also return a Fast Healthcare Interoperability Resources (FHIR) bundle when `fhirVersion` is included in the healthcare task parameters with the supported value `4.0.1`.

## Calling it from the .NET SDK

The healthcare entities operation is asynchronous and batch-oriented. The Microsoft Learn quickstart currently pins the `Azure.AI.TextAnalytics` NuGet package to version `5.2.0`; the NuGet gallery lists newer package versions, and the current .NET API reference shows different long-running operation method names. Keep the sample pinned to the quickstart version, or update the code deliberately when you upgrade the package.

With the quickstart's pinned package, the core loop looks like this:

```csharp
using System;
using System.Collections.Generic;
using Azure;
using Azure.AI.TextAnalytics;

var credentials = new AzureKeyCredential(Environment.GetEnvironmentVariable("LANGUAGE_KEY"));
var endpoint = new Uri(Environment.GetEnvironmentVariable("LANGUAGE_ENDPOINT"));
var client = new TextAnalyticsClient(endpoint, credentials);

var batchInput = new List<string> { "Prescribed 100mg ibuprofen, taken twice daily." };
AnalyzeHealthcareEntitiesOperation operation = await client.StartAnalyzeHealthcareEntitiesAsync(batchInput);
await operation.WaitForCompletionAsync();

await foreach (AnalyzeHealthcareEntitiesResultCollection page in operation.Value)
{
    foreach (AnalyzeHealthcareEntitiesResult doc in page)
    {
        if (doc.HasError)
        {
            Console.WriteLine($"Error: {doc.Error.ErrorCode} - {doc.Error.Message}");
            continue;
        }

        foreach (var entity in doc.Entities)
        {
            Console.WriteLine($"{entity.Text} | {entity.Category} | {entity.NormalizedText}");
        }

        foreach (var relation in doc.EntityRelations)
        {
            Console.WriteLine($"Relation: {relation.RelationType}");
        }
    }
}
```

For this sample, the quickstart output shows `100mg` as `Dosage`, `ibuprofen` as `MedicationName` with normalized text `ibuprofen`, `twice daily` as `Frequency`, and the `DosageOfMedication` and `FrequencyOfMedication` relations.

If your pipeline needs FHIR bundles, follow the FHIR structuring documentation for your chosen access path and pinned SDK version. The REST request pattern adds the FHIR version to the healthcare task parameters:

```json
{
  "kind": "Healthcare",
  "parameters": {
    "fhirVersion": "4.0.1"
  }
}
```

The result includes a `fhirBundle` property for each processed document when the request completes successfully.

## A recommended architecture pattern

For an HHS document-processing pipeline, use a durable, event-driven design:

1. **Ingestion** - documents land in Azure Blob Storage. Use a Government storage endpoint such as `blob.core.usgovcloudapi.net` in Azure Government.
2. **Orchestration** - Event Grid fans documents into Azure Functions or Durable Functions. The hosted Text Analytics for health API supports up to 25 documents per request, subject to the documented character and request-size limits.
3. **De-identification first** - run an appropriate PII or PHI redaction pass before text is persisted to a broader analytics store.
4. **Clinical extraction** - call Text Analytics for health and request FHIR output where downstream systems consume it.
5. **Persistence** - write normalized entities, relations, and UMLS codes to your data platform for cohort analysis and reporting.
6. **Human review** - surface low-confidence or high-impact extractions for case-worker or clinical review rather than auto-acting on them.

Authenticate the Language resource with Microsoft Entra ID and managed identities where possible. Microsoft recommends Microsoft Entra ID authentication with managed identities for Azure resources to avoid storing credentials in cloud applications. Assign the appropriate Azure RBAC role, such as Cognitive Services User, to the managed identity. If you must use keys, store them in Azure Key Vault, rotate them, and restrict access with RBAC and network rules.

## PHI de-identification

Azure Language's PII detection capability identifies, classifies, and redacts sensitive data across text, conversations, and native documents. It has three feature types:

- **Text PII** - synchronous, string-based processing for fields, prompts, logs, and other text values. The [feature-specific documentation](https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/how-to/redact-text-pii) lists stable `2026-05-01` as generally available and `2026-05-15-preview` as preview.
- **Conversation PII** - asynchronous, transcript-aware processing for contact center and intake-call transcripts. Current feature-specific documentation distinguishes GA English support from preview API and model paths. Use the GA path for production English workloads, and treat preview API versions, preview models, and preview-only options as preview unless the documentation changes before deployment.
- **Document-based PII** - asynchronous native-document processing for `.pdf`, `.docx`, and `.txt` files. The [document-based PII overview](https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/document-based-pii-overview) and [native-document how-to](https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/how-to/redact-document-pii) currently mark this capability as preview, so treat it as preview unless the feature-specific documentation has moved it to GA by the time you deploy.

For compliance workflows where you must produce a redacted artifact that still resembles the original, Document-based PII can be a strong fit after you accept the preview terms and risk profile. For production workloads that cannot use preview features, use generally available API versions and avoid mixing request payloads across API versions.

## Keeping data on your infrastructure with containers

When data governance rules will not allow calling a remote endpoint for clinical text, Text Analytics for health is available as a Docker container. For English, Spanish, Italian, French, German, and Portuguese container workloads, the language support page documents the `latin` tag. For Hebrew, use the documented `semitic` tag.

```bash
docker pull mcr.microsoft.com/azure-cognitive-services/textanalytics/healthcare:latin

docker run --rm -it -p 5000:5000 --cpus 6 --memory 12g mcr.microsoft.com/azure-cognitive-services/textanalytics/healthcare:latin Eula=accept rai_terms=accept Billing={ENDPOINT_URI} ApiKey={API_KEY}
```

The container still connects to Azure to send billing metering, but the content of your requests and responses is not visible to Microsoft and is not used for training. Minimum host specs for one document per request are 4 cores and 12 GB of memory; 6 cores and 12 GB are recommended. Once running, it exposes a local REST endpoint:

```bash
curl -X POST 'http://localhost:5000/text/analytics/v3.1/entities/health' --header 'Content-Type: application/json' --header 'accept: application/json' --data-binary @example.json
```

There is also a built-in visualization UI at `http://localhost:5000/demo`. You can host the container on Azure Kubernetes Service, Azure Container Instances, a Kubernetes cluster deployed to Azure Stack, or Web App for Containers.

## Azure Government and compliance considerations

Azure and Azure Government maintain FedRAMP High Provisional Authorizations to Operate for in-scope services. That is a strong assurance signal, but it is not an automatic authorization for an agency application. Agencies still need to validate the specific services, regions, configurations, and application controls in their own authorization process.

Azure Government adds contractual commitments for US public sector agencies: customer data stored in the United States and potential access to systems processing customer data limited to screened US persons. In Azure Government, Text Analytics uses the `cognitiveservices.azure.us` endpoint suffix rather than `cognitiveservices.azure.com`. Confirm Government endpoints with `az cloud show --name AzureUSGovernment` or `Get-AzEnvironment -Name AzureUSGovernment`.

Not every Azure Language feature is available in every region or cloud at the same time. Validate the specific feature and region against the [Azure Language region support](https://learn.microsoft.com/en-us/azure/ai-services/language-service/concepts/regional-support) page and [Products available by region](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/). Where a cloud-hosted feature is not yet available in your target Government region, the self-hosted container can be a practical bridge.

For HIPAA workloads, confirm that the Azure services you use are in scope for Microsoft's HIPAA Business Associate Agreement and that your own application controls meet HIPAA and HITECH obligations. Microsoft documentation is explicit that a BAA supports compliance, but does not automatically make an application compliant.

## Why This Matters for Government

HHS agencies are accountable for outcomes that depend on understanding what is actually in a case file, not just that a file exists. Structuring that text changes the economics:

- **Better cohort visibility** - normalized UMLS codes let analysts ask population-health and program-eligibility questions across documents that were previously opaque.
- **Faster, more consistent processing** - automating concept and relation extraction reduces the abstraction burden on scarce clinical and case-work staff.
- **Privacy by design** - PII and PHI redaction helps agencies share and analyze data with sensitive identifiers removed, supporting minimum-necessary principles.
- **Interoperability** - FHIR-structured output can support downstream health data exchange patterns instead of creating another data silo.
- **Government-cloud assurance** - FedRAMP High P-ATOs for in-scope services, US data residency commitments, and US-person access controls in Azure Government align with the compliance posture many public sector agencies already operate under.

The guardrails are as important as the capability. Keep a human in the loop, treat output as assistive rather than authoritative, redact PHI early, and validate region, API version, preview status, and compliance coverage before production.

### Further reading

- [What is Text Analytics for health?](https://learn.microsoft.com/en-us/azure/ai-services/language-service/text-analytics-for-health/overview)
- [Text Analytics for health quickstart](https://learn.microsoft.com/en-us/azure/ai-services/language-service/text-analytics-for-health/quickstart)
- [FHIR structuring in Text Analytics for health](https://learn.microsoft.com/en-us/azure/ai-services/language-service/text-analytics-for-health/concepts/fhir)
- [Use Text Analytics for health containers](https://learn.microsoft.com/en-us/azure/ai-services/language-service/text-analytics-for-health/how-to/use-containers)
- [PII detection overview](https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/overview)
- [Conversation PII overview](https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/conversation-pii-overview)
- [Document-based PII overview](https://learn.microsoft.com/en-us/azure/ai-services/language-service/personally-identifiable-information/document-based-pii-overview)
- [Compare Azure Government and global Azure](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure)
- [FedRAMP and Azure](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-fedramp)
- [HIPAA and Azure](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-hipaa-us)
