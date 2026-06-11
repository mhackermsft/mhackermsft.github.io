---
title: 'Microsoft Entra Server Principals on Azure SQL Database: Retiring SQL Logins for Identity-First Database Access'
date: 2026-06-11T18:33:36+00:00
author: Mike Hacker
tags:
- Security
- Data Platform
- How To
categories:
- Security
- Data Platform
summary: Microsoft Entra server principals are generally available for Azure SQL Database, enabling passwordless, centrally governed database access with server-level logins, Microsoft Entra-only authentication, and infrastructure-as-code guardrails.
draft: false
image_prompt: A secure Azure SQL Database architecture diagram connected to Microsoft Entra ID, showing identity governance, passwordless access, policy guardrails, and government compliance controls.
image: cover.png
audio: audio.mp3
---

Database credentials remain one of the most stubborn sources of risk in government IT. A connection string with a SQL username and password gets copied into a config file, pasted into a runbook, shared in a ticket, and forgotten in a repository. For agencies pursuing Zero Trust, every one of those static secrets is a liability that lives outside identity governance, multifactor authentication, and Conditional Access.

Microsoft Entra server principals for Azure SQL Database close much of that gap. The [Microsoft Entra server principals documentation](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-logins), last updated May 15, 2026, states that Microsoft Entra logins in the virtual `master` database are generally available for Azure SQL Database, Azure SQL Managed Instance, and SQL Server 2022 and later. Azure Synapse Analytics support is listed separately as public preview.

For state and local agencies, the practical result is important: Azure SQL Database can use server-level Microsoft Entra logins, database users mapped to those logins, fixed server roles, and Microsoft Entra-only authentication. That makes passwordless, identity-governed database access a realistic operating model rather than a future-state aspiration.

## The shift: from contained users to server principals

For years, Azure SQL Database supported Microsoft Entra authentication primarily through contained database users created inside each user database with `CREATE USER ... FROM EXTERNAL PROVIDER`. That model is still supported and still useful, especially for tightly scoped database-only access.

Server principals add a server-level identity object. You create a Microsoft Entra login in the virtual `master` database, then create users in one or more databases from that login. The pattern is familiar to SQL Server administrators, but the credential is a Microsoft Entra identity instead of a password stored in SQL.

The Microsoft Learn documentation lists the benefits directly: support for Azure SQL Database fixed server roles, support for multiple Microsoft Entra users in special roles such as `loginmanager` and `dbmanager`, closer functional parity between SQL logins and Microsoft Entra logins, support for geo-replicas, and integration with Microsoft Entra-only authentication.

## Creating a server-level login

The first Microsoft Entra login must be created by the Microsoft Entra admin configured on the logical server. After that, a principal with the `loginmanager` role can create additional logins. Connect to the virtual `master` database and run:

```sql
CREATE LOGIN [data-eng-team] FROM EXTERNAL PROVIDER;
GO

SELECT name, type_desc, type, is_disabled
FROM sys.server_principals
WHERE type_desc LIKE 'external%';
```

The login name can represent a Microsoft Entra user, group, or application in the same tenant used by Azure SQL. Newly created Microsoft Entra logins in `master` are granted `VIEW ANY DATABASE` by default.

Service principals can share display names in a tenant, which can make a name ambiguous. The `WITH OBJECT_ID` extension resolves that by binding the login to a specific Microsoft Entra object:

```sql
CREATE LOGIN [contoso-etl-app]
FROM EXTERNAL PROVIDER
WITH OBJECT_ID = 'aaaaaaaa-0000-1111-2222-bbbbbbbbbbbb';
```

Use that pattern when the display name is not unique or when you want the login definition to be explicit about the Microsoft Entra object it represents.

## Server-level and database-level access patterns

With the login in place, you have two supported patterns.

**Pattern 1 - Login-mapped users.** Create the login in `master`, then create a user from that login in each user database that needs access:

```sql
CREATE USER [data-eng-team] FROM LOGIN [data-eng-team];
ALTER ROLE db_datareader ADD MEMBER [data-eng-team];
```

This mirrors the classic login-to-user model. One server-level identity can be referenced by multiple databases, and it can participate in Azure SQL Database fixed server roles when the login is a supported user or application login.

**Pattern 2 - Contained database users.** The older syntax remains valid:

```sql
CREATE USER [data-eng-team] FROM EXTERNAL PROVIDER;
```

That creates a contained user with no associated server login. It is still a good fit when you want access scoped to one database and do not need server-role participation.

A subtle but useful detail from the documentation: a login-based Microsoft Entra user has an 18-byte SID with an `AADE` suffix, while a contained Microsoft Entra user has a 16-byte SID. That suffix is how the engine distinguishes the two principal types. If you write reconciliation queries across `sys.server_principals` and `sys.database_principals`, account for that difference.

Server roles are where the server-principal pattern becomes especially useful. In the virtual `master` database, you can add an eligible user or application login to a fixed server role:

```sql
ALTER SERVER ROLE ##MS_DefinitionReader## ADD MEMBER [contoso-etl-app];
```

Azure SQL Database server roles are not supported for Microsoft Entra group logins. Assign those roles to supported user or application logins instead.

## Disabling a user or application login centrally

Because a user or application server principal is a real login, you can disable it in one place and block every database user mapped to it:

```sql
ALTER LOGIN [contoso-etl-app] DISABLE;

DBCC FLUSHAUTHCACHE;
DBCC FREESYSTEMCACHE('TokenAndPermUserStore') WITH NO_INFOMSGS;
```

The cache flush commands make enable, disable, permission, and server-role membership changes take effect immediately for new authorization checks when run with the required permissions in the affected database context. Existing open connections are not terminated automatically, so operational runbooks should handle active sessions when immediate cutoff is required.

Do not use this pattern for Microsoft Entra group logins. Microsoft documents that `ALTER LOGIN ... DISABLE` is not supported for Microsoft Entra groups. For group-based access, govern membership in Microsoft Entra ID and remove SQL role memberships or mapped users when the access model changes.

The same login capability supports geo-replica access patterns. Microsoft documents a scenario where an Entra login is allowed on a geo-replica while connection is denied on the primary.

## Going fully passwordless with Microsoft Entra-only authentication

Server principals are the foundation; Microsoft Entra-only authentication is the control that blocks SQL password authentication. The [Microsoft Entra-only authentication documentation](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-only-authentication) states that enabling the feature disables SQL authentication at the server or managed instance level, including connections from the SQL server admin, SQL logins, and SQL users. Existing SQL logins are not deleted, but they cannot connect while the feature is enabled.

You must set the Microsoft Entra admin before enabling Microsoft Entra-only authentication. With Azure CLI 2.14.2 or later:

```bash
az sql server ad-only-auth enable \
  --resource-group myresource \
  --name myserver

az sql server ad-only-auth get \
  --resource-group myresource \
  --name myserver
```

With Az.Sql 2.10.0 or later:

```powershell
Enable-AzSqlServerActiveDirectoryOnlyAuthentication `
  -ServerName myserver `
  -ResourceGroupName myresource

Get-AzSqlServerActiveDirectoryOnlyAuthentication `
  -ServerName myserver `
  -ResourceGroupName myresource
```

Because enabling the feature blocks rather than deletes SQL logins, agencies can pilot it on non-production logical servers, validate token-based application connections, and then move the pattern into production change control.

## Provisioning identity-first SQL with Bicep

Infrastructure as code is where this becomes durable. The current stable ARM and Bicep references for `Microsoft.Sql/servers@2025-01-01` still require `administratorLoginPassword` when a logical server is created. The secure pattern is to provide the required SQL admin fields as deployment-time properties, set a Microsoft Entra admin, and enable Microsoft Entra-only authentication so SQL authentication is not usable after provisioning.

```bicep
param serverName string
param location string = resourceGroup().location
param aadAdminLogin string
param aadAdminObjectId string
param tenantId string = subscription().tenantId
param sqlAdministratorLogin string = 'sqladminuser'

@secure()
param sqlAdministratorPassword string

resource sqlServer 'Microsoft.Sql/servers@2025-01-01' = {
  name: serverName
  location: location
  properties: {
    administratorLogin: sqlAdministratorLogin
    administratorLoginPassword: sqlAdministratorPassword
    version: '12.0'
    administrators: {
      administratorType: 'ActiveDirectory'
      principalType: 'Group'
      login: aadAdminLogin
      sid: aadAdminObjectId
      tenantId: tenantId
      azureADOnlyAuthentication: true
    }
  }
}
```

Setting the admin to a Microsoft Entra group rather than a single user is the recommended operational pattern. Group membership can be governed in Microsoft Entra ID, so onboarding and offboarding a DBA does not require a SQL change.

For an existing server where the Microsoft Entra admin is already configured, use the dedicated child resource to manage Microsoft Entra-only authentication:

```bicep
param serverName string

resource sqlServer 'Microsoft.Sql/servers@2025-01-01' existing = {
  name: serverName
}

resource entraOnlyAuth 'Microsoft.Sql/servers/azureADOnlyAuthentications@2025-01-01' = {
  parent: sqlServer
  name: 'Default'
  properties: {
    azureADOnlyAuthentication: true
  }
}
```

Microsoft also documents using Azure Policy to require Microsoft Entra-only authentication for new Azure SQL logical servers. Pair the Bicep pattern with policy assignment so identity-first access becomes a guardrail, not just a deployment preference.

## Why This Matters for Government

State and local agencies operate under continuous monitoring expectations, public-records scrutiny, constrained modernization budgets, and formal control frameworks. Static database credentials work against all of those goals.

- **Passwordless access removes a common secret path.** When SQL authentication is disabled, there is no SQL password to leak through a config file, screenshot, deployment variable, or former employee's notes. Microsoft Entra authentication can be governed with Conditional Access and multifactor authentication.
- **Centralized governance improves offboarding.** Mapping database access to Microsoft Entra groups lets agencies manage access through identity governance. Removing a user from a group, or disabling a supported user or application login, can remove mapped database access without touching every database separately.
- **Audit evidence becomes easier to correlate.** Microsoft Entra sign-in logs record tenant sign-ins, including user, service principal, and managed identity sign-ins. Azure SQL auditing records database events to destinations such as Azure Storage, Log Analytics, or Event Hubs. Together, they help agencies correlate authentication and database activity for control evidence.
- **Automation can use application identities.** ETL jobs, reporting services, and deployment pipelines can use Microsoft Entra applications or managed identities rather than embedded SQL passwords. For production workloads, Microsoft recommends managed identities because they eliminate developer-managed credentials.
- **Controls align with public-sector frameworks.** FedRAMP is based on NIST SP 800-53 controls, and Microsoft documents Azure and Azure Government FedRAMP authorizations for in-scope services. Identity-first database access supports least privilege, stronger authentication, and clearer accountability across those control conversations.

For agencies using Azure Government, validate the deployment details in that cloud before standardizing templates. Microsoft lists Azure SQL Database and Microsoft Entra ID endpoints for Azure Government, but Azure Government guidance also notes that feature configurations can differ from global Azure and that service availability should be checked by region. CLI, PowerShell, portal, and ARM API behavior should be confirmed in the target cloud and tenant.

## Getting started

1. Set a Microsoft Entra group as the Azure SQL admin on a non-production logical server.
2. Create several Microsoft Entra server principals with `CREATE LOGIN ... FROM EXTERNAL PROVIDER`.
3. Map login-based users into user databases with `CREATE USER ... FROM LOGIN`.
4. Test application connectivity with Microsoft Entra tokens.
5. Enable Microsoft Entra-only authentication and confirm SQL logins are blocked.
6. Codify the pattern in Bicep and enforce it with Azure Policy across subscriptions.

Microsoft Entra server principals bring Azure SQL Database closer to the server-level identity model that government database teams already understand, while keeping authentication under Microsoft Entra governance. Start with the [Tutorial: Create and utilize Microsoft Entra server logins](https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-azure-ad-logins-tutorial) on Microsoft Learn.
