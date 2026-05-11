---
title: 'Least Privilege AI Agents: Securing Identity-Scoped AI with OAuth2 and Zero Trust'
date: 2026-05-11T19:34:29+00:00
author: Mike Hacker
tags:
- AI
- Security
- How To
categories:
- Security
summary: How to build secure, identity-scoped AI agents using OAuth2 token patterns, delegated permissions, and Zero Trust architecture to enforce least privilege in government AI deployments.
draft: false
image_prompt: A layered shield diagram showing concentric rings of security around an AI agent icon at the center, with OAuth tokens, identity badges, and lock symbols at each layer, rendered in a clean government-professional style with blue and gold tones
image: cover.png
audio: audio.mp3
---

As government organizations accelerate their adoption of AI agents - from citizen-facing support bots to internal data analysis assistants - a critical security question emerges: what can these agents actually access, and on whose behalf?

The answer lies at the intersection of OAuth2 token patterns, delegated permissions, and Zero Trust architecture. In this post, we will walk through how to build AI agents that operate under the principle of least privilege, ensuring they can only access exactly what a user has authorized, nothing more.

## The Overprivileged Agent Problem

AI agents are fundamentally different from traditional applications. A conventional web app requests a fixed set of permissions at login time. An AI agent, by contrast, makes autonomous decisions about which APIs to call, which data to retrieve, and which actions to take - all based on natural language instructions from users.

This autonomy creates a dangerous surface area. An overprivileged AI agent that gets compromised (or simply hallucinates a bad decision) can access data far beyond what any single user interaction requires. Microsoft's Zero Trust documentation is clear on the risk: applications have been found to [fully utilize only 10% of their granted permissions](https://learn.microsoft.com/en-us/security/zero-trust/develop/overprivileged-permissions), meaning 90% of access rights represent unnecessary attack surface.

For government agencies handling sensitive constituent data, tax records, law enforcement information, or public health data, overprivileged agents are not just a security risk - they are a compliance liability.

## Zero Trust Principles for AI Agents

Microsoft's [Zero Trust framework](https://learn.microsoft.com/en-us/security/zero-trust/develop/overview) prescribes three guiding principles that map directly to AI agent security:

- **Verify explicitly**: Every agent request must carry verifiable identity context - both the user's identity and the agent's identity.
- **Use least privilege access**: Agents should receive the narrowest possible scopes, with short-lived tokens that reduce to zero standing privilege.
- **Assume breach**: Design the system so that a compromised agent cannot escalate privileges or access data outside its granted scope.

Applying these principles to AI agents requires rethinking how tokens flow through the system.

## OAuth2 Token Architecture for AI Agents

The core pattern for securing AI agents involves multiple layers of token exchange, where each layer narrows the scope of access. Here is how the architecture works end to end.

### Layer 1: User Authentication and Delegated Permissions

The user authenticates through a standard OAuth2 authorization code flow with PKCE. The resulting access token carries [delegated permissions](https://learn.microsoft.com/en-us/entra/identity-platform/delegated-access-primer) - scopes that represent the intersection of what the application is allowed to do and what the user is allowed to do.

This is critical: delegated permissions mean the agent cannot access anything the signed-in user cannot personally access. Even if the agent holds a `Files.Read.All` scope, it can only read files that the specific user can personally access, not files belonging to other users in the organization.

In Microsoft Entra ID, you register your agent application and configure only the [delegated permissions](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview) it truly needs:

```bash
# Register the agent app with minimal delegated permissions
az ad app create \
  --display-name "CityServicesAgent" \
  --sign-in-audience AzureADMyOrg \
  --required-resource-accesses @permissions.json
```

Where `permissions.json` specifies only the scopes your agent requires:

```json
[
  {
    "resourceAppId": "00000003-0000-0000-c000-000000000000",
    "resourceAccess": [
      {
        "id": "e1fe6dd8-ba31-4d61-89e7-88639da4683d",
        "type": "Scope"
      }
    ]
  }
]
```

### Layer 2: Token Exchange for Scope Downscoping

When an AI agent calls backend services like MCP (Model Context Protocol) servers or other APIs, the gateway should exchange the user's broad token for a narrower, audience-specific token. This follows the [OAuth2 Token Exchange (RFC 8693)](https://datatracker.ietf.org/doc/html/rfc8693) pattern:

```
POST /oauth/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:token-exchange
&subject_token=<incoming_opaque_token>
&subject_token_type=urn:ietf:params:oauth:token-type:access_token
&scope=records/read
&audience=https://api.agency.gov
```

This exchange produces a new token scoped exclusively to the API the agent needs to call, with a short lifetime. If the agent is compromised at this point, the blast radius is limited to read-only access to a single API.

### Layer 3: Opaque Tokens for External Agents, JWTs for Internal Services

A security best practice highlighted in both Microsoft and Curity documentation is to issue **opaque tokens** to external-facing AI agents. Opaque tokens contain no readable claims, so even if an agent is compromised or a malicious MCP client intercepts the token, no sensitive identity information is leaked.

At the API gateway boundary, the opaque token is exchanged for a JWT containing the full claims set that the backend service needs for authorization. This is known as the "phantom token" pattern:

```
[AI Agent] --opaque token--> [API Gateway] --JWT--> [Backend API/MCP Server]
```

The backend API then validates the JWT and enforces fine-grained authorization:

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = configuration.Issuer;
        options.Audience = configuration.Audience;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidAlgorithms = new[] { configuration.Algorithm },
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("scope", policy =>
        policy.RequireAssertion(context =>
            context.User.HasClaim(claim =>
                claim.Type == "scope" &&
                claim.Value.Split(' ').Any(c => c == "records/read")
            )
        )
    );
});
```

### Layer 4: Agent Identity with Foundry Agent Service

Microsoft's [Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-services/agents/overview) provides a fully managed platform for building, deploying, and scaling AI agents with built-in enterprise identity. Each agent can have a dedicated Microsoft Entra identity, enabling:

- **Scoped access to resources**: Agents authenticate to Azure services using managed identities or workload identities without sharing credentials.
- **On-Behalf-Of (OBO) authentication**: Agents pass through the user's identity context when calling downstream MCP servers or APIs.
- **RBAC enforcement**: Fine-grained Azure role assignments control which agents can access which resources.

```csharp
// Connect to Foundry Agent Service with managed identity
PersistentAgentsClient client = new(
    projectEndpoint,
    new DefaultAzureCredential());

// Create an agent with specific model and instructions
PersistentAgent agent = client.Administration.CreateAgent(
    model: "gpt-4.1-mini",
    name: "records-agent",
    instructions: "You are a read-only records assistant",
    tools: mcpTools.ToArray()
);
```

## Access Token Design: What Your Tokens Should Contain

The access token is the single most important security artifact in your AI agent architecture. A well-designed token clearly communicates security context to every API in the chain:

```json
{
  "jti": "31b921b8-b166-4173-b633-7480bab89456",
  "exp": 1762337303,
  "nbf": 1762336403,
  "scope": "records/read",
  "iss": "https://login.agency.gov/oauth/v2",
  "sub": "jane.doe@agency.gov",
  "aud": "https://api.agency.gov",
  "iat": 1762336403,
  "client_type": "ai-agent",
  "client_assurance_level": 1,
  "region": "USA"
}
```

Key design decisions:

- **Short-lived expiration** (15 minutes or less): Ensures zero standing privilege. The agent must re-authenticate frequently.
- **`client_type: ai-agent`**: Allows APIs to apply agent-specific authorization policies.
- **Narrow `scope`**: Only the specific operations this agent needs for this request.
- **`audience` restriction**: The token is only valid for the specific API it targets.

## Auditing Agent Access

Least privilege is only half the equation. You also need visibility into what agents are doing. Route agent requests through an API gateway that captures structured audit logs:

```json
{
  "log_type": "audit",
  "time": "2026-05-11T15:18:09Z",
  "target_path": "/api/records",
  "target_method": "GET",
  "client_id": "city-services-agent",
  "scope": "records/read",
  "delegation_id": "f8b69837-1d3e-4c8a-886f-82923c35955a",
  "user_id": "178",
  "region": "USA"
}
```

Aggregating these logs into Azure Monitor or Microsoft Sentinel enables security teams to detect anomalous agent behavior: unusual scope requests, access patterns outside business hours, or agents attempting to reach APIs outside their designated audience.

## MCP Authorization: The AI Protocol Layer

The [Model Context Protocol (MCP)](https://modelcontextprotocol.io) has emerged as the standard for connecting AI agents to data sources and tools. The MCP authorization specification builds on OAuth 2.1, providing:

- **Dynamic client registration**: Agents can register with MCP servers at runtime.
- **Scope-based tool access**: Each MCP tool (API endpoint) can require specific OAuth scopes.
- **Token exchange at each hop**: As an agent moves from one MCP server to another, tokens are exchanged to match the audience of each server.

This maps naturally to government scenarios where different departments expose different MCP servers - a records MCP server, a permits MCP server, a finance MCP server - each with its own audience and scope requirements.

## Why This Matters for Government

Government organizations face unique pressures that make least-privilege AI agent security non-negotiable:

**Regulatory compliance**: CJIS, FedRAMP, and StateRAMP all require demonstrable access controls and audit trails. An AI agent that operates with broad, unscoped permissions cannot pass these compliance reviews. Delegated permissions with token exchange provide the auditable, scoped access that compliance frameworks demand.

**Constituent data protection**: City and county agencies hold sensitive data: court records, social services case files, tax assessments, and law enforcement data. An overprivileged AI agent in a citizen services portal could expose records across departments. The phantom token pattern ensures that even if the agent-facing token is intercepted, it reveals nothing about the user or the data it can access.

**Multi-agency and cross-jurisdictional scenarios**: State agencies often need agents that access data across multiple departments or jurisdictions. Token exchange enables agents to carry user identity context while narrowing permissions at each boundary, ensuring that a state-level agent accessing county records receives only county-scoped access.

**Azure Government availability**: Organizations using [Azure Government](https://learn.microsoft.com/en-us/azure/azure-government/) should note that Foundry Agent Service availability in government regions may differ from commercial Azure. Verify service availability in your target region and consider hybrid architectures where the agent runtime runs in Azure Government while identity services use Microsoft Entra ID, which is available across both commercial and government clouds.

## Getting Started: Implementation Checklist

1. **Register your agent as a confidential client** in Microsoft Entra ID with only the delegated permissions it needs.
2. **Configure token exchange** at your API gateway to downscope tokens before they reach backend services.
3. **Issue opaque tokens** to agent-facing endpoints; translate to JWTs at the gateway boundary.
4. **Set token lifetimes to 15 minutes or less** to enforce zero standing privilege.
5. **Enable structured audit logging** at the API gateway to capture agent identity, scope, and target for every request.
6. **Deploy agents with managed identities** using Foundry Agent Service to eliminate credential management.
7. **Implement MCP authorization** with scope-per-tool granularity for each data source your agent accesses.

The era of AI agents acting on behalf of government users is here. Building them with least privilege, Zero Trust identity, and proper OAuth2 token architecture is not optional - it is the foundation that makes everything else trustworthy.

## References

- [Microsoft Zero Trust Developer Guide](https://learn.microsoft.com/en-us/security/zero-trust/develop/overview)
- [Delegated Access in Microsoft Identity Platform](https://learn.microsoft.com/en-us/entra/identity-platform/delegated-access-primer)
- [Permissions and Consent Overview](https://learn.microsoft.com/en-us/entra/identity-platform/permissions-consent-overview)
- [Reduce Overprivileged Permissions](https://learn.microsoft.com/en-us/security/zero-trust/develop/overprivileged-permissions)
- [Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-services/agents/overview)
- [Curity: Design MCP Authorization for APIs](https://curity.io/resources/learn/design-mcp-authorization-apis/)
- [Curity: API Security Best Practices for AI Agents](https://curity.io/resources/learn/api-security-best-practice-for-ai-agents/)
- [Curity: Browserless OAuth for AI Agents](https://curity.io/resources/learn/browserless-oauth-ai-agents/)
- [OAuth 2.0 Token Exchange (RFC 8693)](https://datatracker.ietf.org/doc/html/rfc8693)
