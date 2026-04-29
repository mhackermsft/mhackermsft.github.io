---
title: High-Performance Distributed Caching with .NET and PostgreSQL on Azure
date: 2026-04-29T13:14:38+00:00
author: Mike Hacker
tags:
- App Modernization
- Data Platform
- How To
categories:
- Technical
summary: A deep technical walkthrough for building high-performance distributed caching layers using .NET's IDistributedCache interface backed by Azure Database for PostgreSQL, with architecture patterns, code examples, and Bicep deployment templates relevant to government agencies.
draft: false
image_prompt: A massive bronze gear interlocked with a crystalline prism, both resting on a polished marble surface. Warm golden light streams through the prism, casting prismatic reflections across the gear's teeth. Behind them, a series of interconnected copper pipes and valves extends into soft fog, suggesting an intricate plumbing system. The scene is lit with dramatic cinematic side-lighting against a deep navy background. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
audio: audio.mp3
---

Government applications that serve residents - permit portals, benefits eligibility checkers, GIS lookups, licensing systems - share a common performance bottleneck: repeated database queries for data that changes infrequently. A well-designed caching layer can cut response times by 50-90% and dramatically reduce database load. While Redis is the go-to caching solution for many, not every agency wants to introduce a new service into their architecture. If you are already running Azure Database for PostgreSQL, you can now use it as a distributed cache backing store with first-class .NET support.

Microsoft's [`Microsoft.Extensions.Caching.Postgres`](https://www.nuget.org/packages/Microsoft.Extensions.Caching.Postgres) NuGet package implements the `IDistributedCache` interface using PostgreSQL as the storage engine. This means you can add a distributed caching layer to your .NET applications without provisioning a separate caching service, while still using the same abstraction that works with Redis, SQL Server, or Cosmos DB.

## Architecture Overview

The architecture is straightforward: your .NET application uses the standard `IDistributedCache` interface, which routes cache reads and writes to a dedicated table in your PostgreSQL database. The PostgreSQL Flexible Server handles storage, expiration cleanup, and high availability.

**Key architectural benefits:**

- **Single data platform**: No additional service to manage, monitor, or secure
- **Built-in HA**: Leverage PostgreSQL Flexible Server's zone-redundant high availability (99.99% SLA)
- **Connection pooling**: Built-in PgBouncer handles connection management at scale
- **Microsoft Entra authentication**: No passwords to rotate; use managed identities
- **FedRAMP High**: Azure Database for PostgreSQL is available in Azure Government regions

This pattern works best for:

- Session state in multi-instance web applications
- Caching API responses from upstream services
- Storing computed results (eligibility calculations, report aggregations)
- Rate limiting and throttling metadata
- Reference data that changes infrequently (zip codes, fee schedules, department lists)

## Setting Up the Infrastructure

Let's start with a Bicep template that deploys the PostgreSQL Flexible Server with settings optimized for a caching workload.

### Bicep Deployment Template

```bicep
@description('The name of the PostgreSQL Flexible Server')
param serverName string

@description('Location for all resources')
param location string = resourceGroup().location

@description('Administrator login name')
param administratorLogin string

@secure()
@description('Administrator login password')
param administratorLoginPassword string

@description('PostgreSQL version')
param postgresVersion string = '16'

resource postgresServer 'Microsoft.DBforPostgreSQL/flexibleServers@2024-08-01' = {
  name: serverName
  location: location
  sku: {
    name: 'Standard_D4ds_v5'
    tier: 'GeneralPurpose'
  }
  properties: {
    administratorLogin: administratorLogin
    administratorLoginPassword: administratorLoginPassword
    version: postgresVersion
    storage: {
      storageSizeGB: 128
      autoGrow: 'Enabled'
    }
    backup: {
      backupRetentionDays: 14
      geoRedundantBackup: 'Disabled'
    }
    highAvailability: {
      mode: 'ZoneRedundant'
    }
  }
}

// Enable PgBouncer for connection pooling
resource pgBouncerEnabled 'Microsoft.DBforPostgreSQL/flexibleServers/configurations@2024-08-01' = {
  parent: postgresServer
  name: 'pgbouncer.enabled'
  properties: {
    value: 'true'
    source: 'user-override'
  }
}

output serverFqdn string = postgresServer.properties.fullyQualifiedDomainName
output pgBouncerPort int = 6432
```

Deploy with the Azure CLI:

```bash
az deployment group create \
  --resource-group rg-app-caching \
  --template-file postgres-cache.bicep \
  --parameters serverName='pg-cache-prod' \
               administratorLogin='pgadmin' \
               administratorLoginPassword='<secure-password>'
```

> **Azure Government note**: Azure Database for PostgreSQL Flexible Server is available in US Gov Virginia, US Gov Arizona, and US Gov Texas. Zone-redundant HA is supported in US Gov Virginia. When deploying to Azure Government, the server FQDN uses `postgres.database.usgovcloudapi.net` instead of `postgres.database.azure.com`. See [Azure Government database services](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) for endpoint details.

## Configuring the .NET Application

First, install the NuGet package:

```bash
dotnet add package Microsoft.Extensions.Caching.Postgres
```

### Application Settings

Configure your `appsettings.json` with connection pooling parameters optimized for a caching workload:

```json
{
  "ConnectionStrings": {
    "PostgresCache": "Host=pg-cache-prod.postgres.database.azure.com;Port=6432;Database=appdb;Username=pgadmin;Password=<password>;Pooling=true;MinPoolSize=5;MaxPoolSize=100;Timeout=15;SslMode=Require;"
  },
  "PostgresCache": {
    "SchemaName": "cache",
    "TableName": "distributed_cache",
    "CreateIfNotExists": true,
    "UseWAL": false,
    "ExpiredItemsDeletionInterval": "00:15:00",
    "DefaultSlidingExpiration": "00:20:00"
  }
}
```

Note the connection string points to **port 6432**, which routes through the built-in PgBouncer connection pooler. This is critical for caching workloads, where many short-lived connections would otherwise overwhelm PostgreSQL's process-per-connection model. PgBouncer supports up to 10,000 concurrent client connections with minimal overhead. See [PgBouncer in Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-pgbouncer) for configuration details.

### Service Registration

Register the distributed cache in `Program.cs`:

```csharp
using Microsoft.Extensions.Caching.Distributed;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDistributedPostgresCache(options =>
{
    options.ConnectionString = builder.Configuration
        .GetConnectionString("PostgresCache");
    options.SchemaName = builder.Configuration
        .GetValue<string>("PostgresCache:SchemaName", "public");
    options.TableName = builder.Configuration
        .GetValue<string>("PostgresCache:TableName", "distributed_cache");
    options.CreateIfNotExists = builder.Configuration
        .GetValue<bool>("PostgresCache:CreateIfNotExists", true);
    options.ExpiredItemsDeletionInterval = TimeSpan.FromMinutes(15);
    options.DefaultSlidingExpiration = TimeSpan.FromMinutes(20);
});

var app = builder.Build();
```

The `CreateIfNotExists` option automatically creates the cache table on first use, which simplifies deployment. For production, you may prefer to create the table as part of your database migration pipeline for tighter schema control.

### Implementing the Cache-Aside Pattern

Here is a complete service implementation using the cache-aside pattern, which is the most common caching strategy:

```csharp
using System.Text.Json;
using Microsoft.Extensions.Caching.Distributed;

public class PermitLookupService
{
    private readonly IDistributedCache _cache;
    private readonly IPermitRepository _repository;
    private readonly ILogger<PermitLookupService> _logger;

    public PermitLookupService(
        IDistributedCache cache,
        IPermitRepository repository,
        ILogger<PermitLookupService> logger)
    {
        _cache = cache;
        _repository = repository;
        _logger = logger;
    }

    public async Task<PermitStatus?> GetPermitStatusAsync(
        string permitId, CancellationToken ct = default)
    {
        var cacheKey = $"permit:status:{permitId}";

        // Try cache first
        var cached = await _cache.GetAsync(cacheKey, ct);
        if (cached is not null)
        {
            _logger.LogDebug("Cache hit for {PermitId}", permitId);
            return JsonSerializer.Deserialize<PermitStatus>(cached);
        }

        // Cache miss: query the database
        _logger.LogDebug("Cache miss for {PermitId}", permitId);
        var status = await _repository.GetStatusAsync(permitId, ct);
        if (status is null) return null;

        // Store in cache with expiration policy
        var options = new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1),
            SlidingExpiration = TimeSpan.FromMinutes(15)
        };

        var serialized = JsonSerializer.SerializeToUtf8Bytes(status);
        await _cache.SetAsync(cacheKey, serialized, options, ct);

        return status;
    }

    public async Task InvalidatePermitCacheAsync(
        string permitId, CancellationToken ct = default)
    {
        var cacheKey = $"permit:status:{permitId}";
        await _cache.RemoveAsync(cacheKey, ct);
        _logger.LogInformation(
            "Cache invalidated for {PermitId}", permitId);
    }
}
```

This pattern ensures:

- **Fast reads**: Cached data is returned without hitting the primary database
- **Staleness control**: Sliding expiration keeps hot data fresh; absolute expiration prevents unbounded staleness
- **Explicit invalidation**: When a permit status changes, the cache entry is removed so the next read fetches fresh data

### A Generic Cache Helper

To avoid repeating cache-aside boilerplate across services, consider a reusable extension method:

```csharp
public static class DistributedCacheExtensions
{
    public static async Task<T?> GetOrSetAsync<T>(
        this IDistributedCache cache,
        string key,
        Func<CancellationToken, Task<T?>> factory,
        DistributedCacheEntryOptions options,
        CancellationToken ct = default) where T : class
    {
        var cached = await cache.GetAsync(key, ct);
        if (cached is not null)
            return JsonSerializer.Deserialize<T>(cached);

        var value = await factory(ct);
        if (value is null) return null;

        var bytes = JsonSerializer.SerializeToUtf8Bytes(value);
        await cache.SetAsync(key, bytes, options, ct);
        return value;
    }
}
```

Usage becomes a single call:

```csharp
var status = await _cache.GetOrSetAsync(
    $"permit:status:{permitId}",
    ct => _repository.GetStatusAsync(permitId, ct),
    new DistributedCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1),
        SlidingExpiration = TimeSpan.FromMinutes(15)
    },
    cancellationToken);
```

## Performance Tuning

### PgBouncer Configuration

For caching workloads, tune PgBouncer's server parameters through the Azure Portal or CLI:

```bash
# Increase default pool size for high-throughput caching
az postgres flexible-server parameter set \
  --resource-group rg-app-caching \
  --server-name pg-cache-prod \
  --name pgbouncer.default_pool_size \
  --value 100

# Set transaction pooling mode (default, optimal for short cache ops)
az postgres flexible-server parameter set \
  --resource-group rg-app-caching \
  --server-name pg-cache-prod \
  --name pgbouncer.pool_mode \
  --value transaction

# Increase max client connections
az postgres flexible-server parameter set \
  --resource-group rg-app-caching \
  --server-name pg-cache-prod \
  --name pgbouncer.max_client_conn \
  --value 10000
```

Transaction pooling mode is ideal for caching because each cache operation is a short, independent query. This maximizes connection reuse.

### Write-Ahead Logging Consideration

The `UseWAL` option in the Postgres cache configuration controls whether the cache table uses PostgreSQL's Write-Ahead Logging. Setting `UseWAL = false` (the default) creates the cache table as `UNLOGGED`, which significantly improves write performance because PostgreSQL skips writing changes to the WAL. The trade-off is that unlogged table data is lost after a server crash or unclean shutdown. For caching, this is typically acceptable since cache data is ephemeral by design.

If your cache stores data that is expensive to recompute (complex report aggregations that take minutes), consider setting `UseWAL = true` for durability at the cost of some write performance.

### Cache Key Design

Effective cache key design prevents collisions and enables bulk invalidation:

```csharp
// Hierarchical key structure
$"permits:status:{permitId}"        // Individual permit
$"fees:schedule:{feeType}:{year}"   // Fee schedule by type and year
$"lookup:zipcodes:{stateCode}"      // Reference data by state
$"session:{sessionId}"              // User session data
```

## When to Choose PostgreSQL vs. Redis for Caching

| Factor | PostgreSQL Cache | Azure Cache for Redis |
|--------|-----------------|----------------------|
| Additional service to manage | No | Yes |
| Sub-millisecond latency | No (single-digit ms typical) | Yes |
| Azure Government availability | Yes (3 regions) | Yes (2 regions) |
| Connection pooling built-in | Yes (PgBouncer) | N/A |
| Data structure support | Key-value only | Lists, sets, sorted sets, streams |
| Cost for small workloads | Lower (shared with existing DB) | Higher (dedicated instance) |
| FedRAMP High authorized | Yes | Yes |

Choose PostgreSQL caching when: you already run PostgreSQL, your caching needs are straightforward key-value, and you want to minimize infrastructure complexity. Choose Redis when: you need sub-millisecond latency, advanced data structures, or pub/sub capabilities.

## Why This Matters for Government

Government agencies face a unique combination of constraints: strict compliance requirements (FedRAMP, StateRAMP), limited staffing for infrastructure management, and increasing demand from residents for responsive digital services. The PostgreSQL distributed caching pattern addresses all three.

**Reduced attack surface**: Every additional service in your architecture is another component to patch, monitor, and secure. By using your existing PostgreSQL instance for caching, you eliminate a separate caching service from your Authority to Operate (ATO) boundary documentation.

**Lower total cost of ownership**: Government IT budgets are scrutinized carefully. Consolidating caching into your existing database eliminates the cost of a dedicated Redis instance and reduces operational overhead for teams that are already stretched thin.

**Improved resident experience**: Caching frequently accessed data - fee schedules, permit statuses, eligibility lookup results - means faster page loads for residents interacting with government portals. A reduction from 500ms to 50ms response time can meaningfully improve accessibility, especially for users on slower mobile connections.

**Azure Government compatibility**: Azure Database for PostgreSQL Flexible Server is available in US Gov Virginia, US Gov Arizona, and US Gov Texas, with zone-redundant high availability in US Gov Virginia. The `Microsoft.Extensions.Caching.Postgres` package works identically in both Azure Commercial and Azure Government. Just update your connection string endpoint from `postgres.database.azure.com` to `postgres.database.usgovcloudapi.net`.

**Compliance alignment**: PostgreSQL Flexible Server supports [Microsoft Entra authentication](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-manage-azure-ad-users), eliminating the need to store database passwords. Combined with Azure Private Link and virtual network integration, this meets the zero-trust networking requirements that many state and local agencies are adopting.

## Further Reading

- [Distributed caching in ASP.NET Core (Microsoft Learn)](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/distributed?view=aspnetcore-9.0)
- [Azure Database for PostgreSQL Flexible Server overview](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview)
- [PgBouncer in Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-pgbouncer)
- [Microsoft.Extensions.Caching.Postgres NuGet package](https://www.nuget.org/packages/Microsoft.Extensions.Caching.Postgres)
- [Azure Government database endpoints](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure)
- [Microsoft Entra authentication for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-manage-azure-ad-users)

