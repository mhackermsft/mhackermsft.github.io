---
title: 'Graph API Breaking Change: New Permissions Required for Sensitive Email Properties by December 2026'
date: 2026-03-30T12:31:25+00:00
author: Mike Hacker
tags:
- Security
- App Modernization
- Announcements
- How To
categories:
- Microsoft 365
summary: Microsoft is restricting Graph API updates to sensitive email properties on non-draft messages starting December 31, 2026, requiring the new Mail-Advanced.ReadWrite permission with admin consent.
draft: false
image_prompt: A massive ornate brass padlock being placed onto an oversized sealed envelope with a wax seal, set against a dramatic twilight sky with deep purples and golds. A ring of antique skeleton keys floats around the envelope, each key glowing with a different intensity. The scene conveys controlled access and the protection of sealed correspondence. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
audio: audio.mp3
---

Microsoft recently announced a significant breaking change to the Microsoft Graph API that every government M365 developer and IT administrator needs to know about. Starting **December 31, 2026**, applications that modify sensitive email properties on non-draft messages will be blocked unless they hold the new **Mail-Advanced.ReadWrite** permission. If your organization runs custom integrations, line-of-business applications, or third-party tools that touch Exchange Online email via Graph API, now is the time to audit and update them.

This post breaks down what is changing, which properties are affected, how to migrate your applications, and why this matters for government organizations.

## What Is Changing

Today, any application with the standard `Mail.ReadWrite` permission can update virtually any property on an email message through the [Graph API message update endpoint](https://learn.microsoft.com/en-us/graph/api/message-update?view=graph-rest-1.0&tabs=http). That includes the ability to modify the subject line, body content, and recipient lists on messages that have already been sent and received.

Microsoft is tightening this behavior to enforce a fundamental email integrity principle: **once a message is received, its core content should not change**. Properties like read status, flags, categories, and importance will remain freely updatable with standard permissions. However, sensitive properties, those that fundamentally alter the meaning or provenance of a message, will require elevated permissions on non-draft messages.

The [announcement from the Exchange Team](https://devblogs.microsoft.com/microsoft365dev/graph-api-updates-to-sensitive-email-properties/) was published on March 26, 2026, giving organizations roughly nine months to prepare.

## Which Properties Are Affected

The [Update message documentation](https://learn.microsoft.com/en-us/graph/api/message-update?view=graph-rest-1.0&tabs=http) identifies sensitive properties as those currently annotated with "Updatable only if **isDraft** = true." These include:

| Property | Type | Description |
|---|---|---|
| `subject` | String | The subject line of the message |
| `body` | ItemBody | The HTML or text body content |
| `toRecipients` | Recipient collection | The To: recipients |
| `ccRecipients` | Recipient collection | The Cc: recipients |
| `bccRecipients` | Recipient collection | The Bcc: recipients |
| `from` | Recipient | The mailbox owner and sender |
| `sender` | Recipient | The account used to generate the message |
| `replyTo` | Recipient collection | Reply-to addresses |
| `internetMessageId` | String | The RFC2822 message ID |
| `singleValueExtendedProperties` | Collection | Single-value extended properties |
| `multiValueExtendedProperties` | Collection | Multi-value extended properties |

After enforcement begins, attempting to update any of these properties on a non-draft message without the required elevated permission will result in an API error. **Draft messages will continue to be fully updatable with the standard `Mail.ReadWrite` permission.**

## The New Permission: Mail-Advanced.ReadWrite

Microsoft has introduced a new family of permissions specifically designed for advanced mail operations. To continue modifying sensitive properties on non-draft messages, your application must hold one of the following:

- **Mail-Advanced.ReadWrite** - Delegated permission for modifying sensitive properties on behalf of a signed-in user
- **Mail-Advanced.ReadWrite.All** - Application permission for modifying sensitive properties across all mailboxes without a signed-in user
- **Mail-Advanced.ReadWrite.Shared** - Delegated permission for modifying sensitive properties in shared mailboxes

All three of these permissions **require tenant administrator consent**. This is by design: modifying the content of received email is a privileged operation typically reserved for security tools, compliance workflows, and specialized administrative scenarios.

The new permissions are already available in Microsoft Entra ID today. You do not need to wait until the enforcement date to add them to your app registrations. Microsoft strongly recommends updating your applications as soon as possible.

## Step-by-Step Migration Guide

Here is a practical walkthrough for government IT teams to prepare for this change.

### Step 1: Inventory Your Applications

Identify every application in your tenant that uses the Microsoft Graph Mail API. You can query your Entra ID app registrations to find applications with `Mail.ReadWrite` permissions:

```http
GET https://graph.microsoft.com/v1.0/servicePrincipals?
  $filter=appRoles/any(r:r/value eq 'Mail.ReadWrite')
```

Also check for third-party applications and vendor integrations. Common categories that may be affected include:

- **Email archiving and records management** systems that tag or modify messages
- **Security and threat response** tools that quarantine or annotate suspicious emails
- **Helpdesk and ticketing** systems that update email subjects or bodies for tracking
- **Data loss prevention (DLP)** solutions that redact sensitive content
- **Migration tools** that move and transform messages between mailboxes

### Step 2: Determine Which Apps Actually Need Elevated Permissions

Not every app with `Mail.ReadWrite` modifies sensitive properties. Many only change `isRead`, `categories`, `flag`, or `importance`, which are not affected by this change. Review your application code to determine whether it writes to any of the sensitive properties listed above on non-draft messages.

Apps that only create draft messages, send new messages, or read mail are not affected.

### Step 3: Update App Registrations in Entra ID

For applications that do need to modify sensitive properties on non-draft messages:

1. Navigate to the [Azure portal](https://portal.azure.com) (or [portal.azure.us](https://portal.azure.us) for GCC High environments)
2. Go to **Microsoft Entra ID** > **App registrations**
3. Select the application
4. Under **API permissions**, click **Add a permission**
5. Choose **Microsoft Graph**
6. Search for and add the appropriate `Mail-Advanced.ReadWrite` permission variant
7. Click **Grant admin consent** for your organization

### Step 4: Update Application Code

If your application requests permissions dynamically (common in delegated flows), update the scopes in your authentication code:

```csharp
// Before
var scopes = new[] { "Mail.ReadWrite" };

// After - include the advanced permission
var scopes = new[] { "Mail.ReadWrite", "Mail-Advanced.ReadWrite" };
```

For application-only (daemon) flows using client credentials, ensure `Mail-Advanced.ReadWrite.All` is configured in the app registration. No code change is needed for these flows beyond the Entra ID configuration.

### Step 5: Test in a Non-Production Environment

Before rolling changes to production, validate in a development or staging tenant:

- Confirm that your application can still read and modify draft messages with standard permissions
- Verify that non-draft message updates to sensitive properties succeed with the new `Mail-Advanced.ReadWrite` permission
- Ensure that non-sensitive property updates (flags, categories, read status) continue to work with `Mail.ReadWrite` alone

### Step 6: Plan Your Rollout

Coordinate with your tenant administrators. Since `Mail-Advanced.ReadWrite` requires admin consent, this is not something individual users can approve. Build this into your change management process and allow time for any required security reviews.

## GCC and Government Cloud Considerations

The Graph API [message update endpoint is available](https://learn.microsoft.com/en-us/graph/api/message-update?view=graph-rest-1.0) across all national cloud deployments, including **GCC**, **GCC High**, and **DoD** environments. Government organizations should be aware of the following:

- **GCC tenants** use the standard `https://graph.microsoft.com` endpoint and the `https://portal.azure.com` portal for app registrations
- **GCC High tenants** use `https://graph.microsoft.us` and `https://portal.azure.us`
- **DoD tenants** use `https://dod-graph.microsoft.us` and `https://portal.azure.us`

Microsoft has not indicated that enforcement timelines will differ across cloud environments, so government organizations should plan for the same **December 31, 2026** deadline. When the new permissions become enforceable in GCC High and DoD environments, you will need to ensure your app registrations in those specific clouds have been updated. Permissions granted in a commercial tenant do not carry over to government cloud tenants.

For more information on national cloud deployments, see [Microsoft Graph national cloud deployments](https://learn.microsoft.com/en-us/graph/deployments).

## Why This Matters for Government

Government organizations have heightened requirements for **email integrity, records retention, and chain-of-custody** compliance. This breaking change directly supports those objectives:

- **Records management compliance**: Many state and local government agencies are subject to public records laws that require email messages to be preserved in their original form. Restricting the ability to silently alter received messages aligns with these legal requirements and reduces the risk of accidental or malicious tampering.

- **Security posture improvement**: By requiring elevated, admin-consented permissions for sensitive modifications, Microsoft is adding a meaningful gate that prevents over-permissioned applications from altering email content. This is a win for zero-trust architectures that government agencies are increasingly adopting.

- **Audit and accountability**: The separation between standard mail operations and advanced modifications makes it easier for security teams to identify and monitor which applications have the ability to alter message content, a critical capability for incident response and forensic investigations.

- **Vendor management**: Government procurement teams should proactively reach out to third-party software vendors that integrate with Exchange Online via Graph API. Ask vendors whether their products modify sensitive email properties and whether they have a plan to adopt the new permissions before the deadline.

- **Change management lead time**: Government IT change management processes often require longer approval cycles than private sector organizations. With nine months until enforcement, now is the time to initiate the process, not six months from now when calendar year-end freezes may be in effect.

## Key Dates and Action Items

| Date | Milestone |
|---|---|
| **Now** | Mail-Advanced.ReadWrite permissions are available for use |
| **Q2 2026** | Audit applications, identify affected integrations |
| **Q3 2026** | Update app registrations, test in non-production environments |
| **Q4 2026** | Complete production rollout, coordinate with vendors |
| **December 31, 2026** | Enforcement begins: unauthorized updates to sensitive properties will be blocked |

## Additional Resources

- [Breaking Change Ahead: Graph API Updates to Sensitive Email Properties](https://devblogs.microsoft.com/microsoft365dev/graph-api-updates-to-sensitive-email-properties/) - Official announcement from the Exchange Team
- [Update message - Microsoft Graph API Reference](https://learn.microsoft.com/en-us/graph/api/message-update?view=graph-rest-1.0&tabs=http) - Full documentation on message update operations and property restrictions
- [Microsoft Graph permissions reference](https://learn.microsoft.com/en-us/graph/permissions-reference) - Complete list of Graph API permissions including the new Mail-Advanced variants
- [Microsoft Graph national cloud deployments](https://learn.microsoft.com/en-us/graph/deployments) - Endpoint and registration details for GCC, GCC High, and DoD environments

## Summary

This is not a subtle deprecation notice buried in a changelog. This is a hard enforcement date that will cause API failures for unprepared applications. The good news is that the new permissions are available today, the migration path is straightforward, and you have nine months to prepare. Start your application audit now, coordinate with your vendors, and build the permission updates into your next change management cycle. Your future self (and your compliance team) will thank you.

