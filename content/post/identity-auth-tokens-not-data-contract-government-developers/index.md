---
title: 'Identity: Stop Treating Auth Tokens as a Data Contract - A Guide for Government Developers'
date: 2026-03-20T13:29:48+00:00
author: Mike Hacker
tags:
- Security
- App Modernization
- How To
- Announcements
categories:
- Security
summary: Learn why parsing JWT and OAuth token claims is a dangerous anti-pattern in government systems, and how to build stable, Zero Trust-aligned integrations using Microsoft Entra ID and Azure API Management.
draft: false
image_prompt: A dramatic close-up of a heavy ornate iron lock with a glowing golden keyhole, surrounded by floating metallic gears and interlocking chain links. Rays of cool blue and amber light pierce through from behind, casting long shadows. Several skeleton keys hover at different angles, with one correct key emitting a warm glow as it aligns with the lock. The overall scene conveys access control and authorization - a key is only useful for unlocking a door, not as a container of stored information. Deep shadows and high-contrast cinematic lighting. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
---

A warning landed quietly on the Azure DevOps blog last week, but the implications reach far beyond a single product team. In a post titled [Authentication Tokens Are Not a Data Contract](https://devblogs.microsoft.com/devops/authentication-tokens-are-not-a-data-contract/), the Azure DevOps engineering team delivered a clear message: if your application decodes authentication tokens and reads claims from them, you are building on a foundation that was never meant to support you - and that foundation is about to shift.

For government developers and IT leaders managing integrations across city, county, and state systems, this is not a distant cloud-vendor concern. It is an architectural risk embedded in many of the custom applications, middleware pipelines, and API integrations that run government services today.

## What Is the Anti-Pattern?

JSON Web Tokens (JWTs) are Base64-encoded, which means they appear readable to anyone who pastes one into a tool like [jwt.ms](https://jwt.ms). That readability creates a temptation: instead of calling a proper API to get user or resource data, a developer decodes the token and reads values directly from the payload. The user's name is in `name`. The tenant is in `tid`. The roles are in `roles`. Why make another API call?

Here is why: **those claims are not a contract**.

As Microsoft's own documentation states, access tokens should be treated as opaque strings by client applications. The [Microsoft identity platform documentation](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens) is explicit on this point:

> "Although client applications can receive and use access tokens, they should be treated as opaque strings. The client application shouldn't attempt to validate access tokens."

The token exists to answer one question - is this caller authorized to do this? It is not a user profile API. It is not a schema. It is not a reliable source of application data.

## Why Claims Are Not Stable

Token claims have always been an implementation detail, not a published interface. Microsoft has never formally documented the internal structure of access tokens issued for its own services, and has always reserved the right to change them. Claims can:

- Be renamed or restructured
- Become optional or be removed entirely
- Change format or data type
- Disappear entirely when token encryption is applied

The Azure DevOps team made this concrete: **starting this summer, Azure DevOps will further encrypt authentication tokens**, making the payload unreadable by clients. Any application currently decoding Azure DevOps tokens to extract claims will break when this change takes effect. The team also noted that some changes may arrive before that deadline as token formats continue to evolve.

This is not unique to Azure DevOps. Token formats across the Microsoft identity platform can change, and the v1.0 and v2.0 formats already differ in meaningful ways. The `tid`, `oid`, `sub`, and `email` claims behave differently depending on the audience, the token version, and the registration configuration of the application. Building logic against those claims without using the proper validation and API layers is asking for failure.

## What the Right Pattern Looks Like

The correct approach has three layers.

### 1. Use Tokens Only for Authorization

Acquire the token, pass it in the Authorization header, and let the resource server do the validation. The client application's job ends there. If your application needs data about the calling user - their display name, their group memberships, their department - call the [Microsoft Graph API](https://learn.microsoft.com/en-us/graph/overview) or your own resource API. Those are versioned, documented, and stable contracts.

For Azure DevOps-specific data such as user identities, organization memberships, or project permissions, use the [Azure DevOps REST APIs](https://learn.microsoft.com/en-us/rest/api/azure/devops/?view=azure-devops-rest-7.2) directly rather than inferring that data from token claims.

### 2. Validate Tokens at the Resource Server - The Right Way

If you are building an API that needs to validate incoming tokens, the validation should check:

- The token **signature**, using the public keys published by the identity provider's OpenID Connect discovery endpoint
- The **issuer** (`iss` claim), ensuring it matches your expected Microsoft Entra ID tenant
- The **audience** (`aud` claim), ensuring the token was issued specifically for your API
- The **expiration** (`exp` claim)

This is the only token inspection that is architecturally sound. Use a well-maintained library - such as [Microsoft.Identity.Web](https://learn.microsoft.com/en-us/entra/msal/dotnet/microsoft-identity-web/) for .NET applications - to handle this validation rather than parsing tokens manually. The library abstracts key rotation, version differences, and claim normalization so your code does not have to.

### 3. Use Azure API Management as Your Token Enforcement Layer

[Azure API Management (APIM)](https://learn.microsoft.com/en-us/azure/api-management/api-management-key-concepts) provides a purpose-built policy engine for JWT validation before requests ever reach your backend services. The [`validate-jwt` policy](https://learn.microsoft.com/en-us/azure/api-management/validate-jwt-policy) checks signature, issuer, audience, and required claims at the gateway layer:

```xml
<validate-jwt header-name="Authorization"
              failed-validation-httpcode="401"
              failed-validation-error-message="Unauthorized. Access token is missing or invalid."
              require-expiration-time="true"
              require-signed-tokens="true">
  <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
  <audiences>
    <audience>api://your-api-app-id</audience>
  </audiences>
  <issuers>
    <issuer>https://login.microsoftonline.com/{tenant-id}/v2.0</issuer>
  </issuers>
</validate-jwt>
```

This pattern means your backend APIs receive only pre-validated requests, and authorization logic is centralized. There is no need to decode token payloads in application code at all. APIM is available in both Azure Commercial and Azure Government environments, making this pattern applicable regardless of which cloud your agency operates in.

## Why This Matters for Government

Government systems carry unique constraints that make token parsing especially risky.

**Integration longevity.** Government integrations often run for years or decades without significant code changes. A fragile pattern that breaks when a token format changes can cause cascading failures across systems that residents depend on - from permitting and licensing portals to court case management and benefits delivery.

**Compliance and auditability.** Zero Trust mandates, NIST SP 800-207, and frameworks like [CISA's Zero Trust Maturity Model](https://www.cisa.gov/zero-trust-maturity-model) all require explicit verification at every access request. Extracting authorization logic from unstable token claims rather than verified, auditable API responses undermines the "verify explicitly" principle that is foundational to Zero Trust.

**Hybrid identity complexity.** Most government agencies operate hybrid identity environments - on-premises Active Directory synchronized to Microsoft Entra ID. Claims in tokens issued by hybrid environments can differ from those in cloud-only environments. Claim values may be sourced from on-premises attributes that change during directory cleanup, migration, or schema updates, making them even less reliable as application data.

**Security boundary enforcement.** Tokens issued for one service should never be accepted as proof of authorization for a different service. This is the [confused deputy problem](https://cwe.mitre.org/data/definitions/441.html), and it is a real vulnerability. Proper audience validation - enforced through APIM or library-based validation - prevents tokens from being replayed across APIs they were not issued for.

**GCC tenant considerations.** Agencies operating in M365 GCC environments should be especially careful about relying on token claim content. GCC tenants route through Microsoft Entra Public (`login.microsoftonline.com`), while workloads on Azure Government use a separate authority (`login.microsoftonline.us`). Mismatched authority configurations are a common source of integration failures, and they can cause silent claim differences or outright authentication errors if applications have hardcoded assumptions about token structure.

## Zero Trust Alignment

The [Microsoft Zero Trust guidance for identity](https://learn.microsoft.com/en-us/security/zero-trust/deploy/identity) centers on three principles: verify explicitly, use least privilege access, and assume breach. Token parsing works against all three:

- **Verify explicitly** requires validating every aspect of the request at runtime - not assuming claims contain the right values because they appeared correct last month.
- **Least privilege** requires granting access based on verified, current permissions - not on a snapshot of claims baked into a token at issuance time.
- **Assume breach** means your application logic should not depend on internal structures that could be manipulated or misread.

Treating tokens as opaque, validating signatures and audience claims properly, and using Microsoft Graph or your own resource APIs for user data is the architecture that aligns with Zero Trust. It is also the architecture that survives token format changes.

## Azure Commercial vs. Azure Government: Know Your Endpoints

For teams running workloads across both Azure Commercial and Azure Government clouds - a common pattern for state and local government agencies - there are critical endpoint differences to keep in mind:

| Environment | Authority Endpoint | Microsoft Graph |
|---|---|---|
| Azure Commercial / GCC | `login.microsoftonline.com` | `graph.microsoft.com` |
| Azure Government | `login.microsoftonline.us` | `graph.microsoft.us` |

Applications that hardcode the commercial authority endpoint will fail to validate tokens issued by the government cloud authority, and vice versa. Use [environment-aware configuration](https://learn.microsoft.com/en-us/azure/azure-government/documentation-government-plan-identity) and abstract your authority URL behind a configuration setting rather than a code constant. Libraries like Microsoft.Identity.Web and MSAL support named cloud environments to simplify this.

## An Action Plan for Government IT Teams

If any of this sounds like code your team has written or inherited, here is a practical starting point:

1. **Audit your integrations.** Search codebases for patterns like JWT decode calls, `base64decode`, or claim name strings such as `oid`, `tid`, `upn`, or `roles` being read from token payloads outside of a validated library context.

2. **Replace claim reads with API calls.** For user data, use Microsoft Graph. For authorization data, use your own resource APIs or Microsoft Entra ID's [role-based access control features](https://learn.microsoft.com/en-us/entra/identity/role-based-access-control/overview).

3. **Move token validation to APIM.** Centralize JWT validation using Azure API Management's `validate-jwt` policy and remove redundant token parsing from backend services.

4. **Adopt Microsoft.Identity.Web.** For .NET APIs, this library handles token validation correctly and keeps pace with Microsoft identity platform changes automatically.

5. **Configure for your cloud.** Ensure applications are configured with the correct authority endpoint for the cloud they run in, and that authority URL is externalized to configuration rather than hardcoded.

6. **Test before summer 2026.** The Azure DevOps token encryption change is coming this summer. If your pipelines or integrations reference Azure DevOps tokens, validate they treat those tokens as opaque now.

## Closing Thoughts

Authentication tokens are infrastructure, not data. They are the key in the lock - not the records in the filing cabinet behind it. When the Azure DevOps engineering team says token claims were never a promise, they are speaking for the entire Microsoft identity platform.

Government developers building systems that serve residents deserve integrations that do not break quietly because an upstream team changed an internal field name. The pattern is well-established, the tools are available in both Azure Commercial and Azure Government, and the guidance is unambiguous.

Treat your tokens as opaque. Call the APIs. Build for stability.

---

*Additional resources:*
- [Authentication Tokens Are Not a Data Contract - Azure DevOps Blog](https://devblogs.microsoft.com/devops/authentication-tokens-are-not-a-data-contract/)
- [Microsoft Entra ID access tokens - Microsoft Learn](https://learn.microsoft.com/en-us/entra/identity-platform/access-tokens)
- [validate-jwt policy - Azure API Management](https://learn.microsoft.com/en-us/azure/api-management/validate-jwt-policy)
- [Zero Trust identity deployment guide - Microsoft](https://learn.microsoft.com/en-us/security/zero-trust/deploy/identity)
- [Identity planning for Azure Government - Microsoft Learn](https://learn.microsoft.com/en-us/azure/azure-government/documentation-government-plan-identity)
- [Microsoft.Identity.Web library](https://learn.microsoft.com/en-us/entra/msal/dotnet/microsoft-identity-web/)
- [Microsoft Graph API overview](https://learn.microsoft.com/en-us/graph/overview)
