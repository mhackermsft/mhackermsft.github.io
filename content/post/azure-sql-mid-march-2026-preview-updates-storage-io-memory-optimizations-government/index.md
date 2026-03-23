---
title: 'Azure SQL Mid-March 2026 Preview Updates: Storage, I/O, and Memory Optimizations for Government Database Workloads'
date: 2026-03-23T13:01:34+00:00
author: Mike Hacker
tags:
- Data Platform
- Announcements
- How To
- App Modernization
categories:
- Azure SQL
summary: Azure SQL's new Automatic Index Compaction public preview delivers transparent storage, I/O, and memory gains for government database workloads - no application code changes required.
draft: false
image_prompt: A massive translucent crystal database cylinder being compressed by glowing blue mechanical arms, fragments of scattered data blocks consolidating into a dense perfect structure, dramatic cinematic lighting with deep blue and white Azure tones against a dark background. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
---

Mid-March 2026 brought a cluster of noteworthy Azure SQL updates, and one feature in particular stands out for government database administrators: **Automatic Index Compaction**, now in public preview. Alongside a wave of engine-level improvements that reached general availability over the past several months, this release represents a meaningful shift in how Azure SQL manages itself - doing more of the maintenance heavy lifting on your behalf, and doing it continuously, quietly, and efficiently.

For state and local government IT teams that manage legacy databases, citizen-facing transactional systems, or large repositories of case management and permit data, these updates offer something genuinely practical: measurable performance and cost improvements without requiring changes to existing application code.

## What Is Automatic Index Compaction?

In any active database, rows are constantly being inserted, updated, and deleted. Over time, this activity leaves partially filled data pages scattered across the index structure - a phenomenon known as index fragmentation. Traditional remedies involve scheduled index rebuild or reorganize jobs that run during maintenance windows, consume significant CPU and I/O, require extra free space in data files, and still only address fragmentation at a point in time.

Automatic Index Compaction, [now in public preview for Azure SQL Database and Azure SQL Managed Instance](https://techcommunity.microsoft.com/blog/azuresqlblog/stop-defragmenting-and-start-living-introducing-auto-index-compaction/4500089) (with the Always-up-to-date update policy), takes a fundamentally different approach. Rather than periodically rebuilding entire indexes, it runs continuously in the background as part of the existing [Persistent Version Store (PVS)](https://learn.microsoft.com/en-us/azure/azure-sql/accelerated-database-recovery) cleaner process. As pages are visited during routine cleanup, the compaction process consolidates rows from sparsely-filled pages into denser, fuller pages - and then deallocates the now-empty pages.

The result is that page density stays consistently high, and query workloads read fewer pages to return the same data.

You enable it with a single T-SQL command:

```sql
ALTER DATABASE [your-database-name] SET AUTOMATIC_INDEX_COMPACTION = ON;
```

That is all the code change required.

## The Performance Numbers Are Significant

Microsoft's own testing demonstrates the impact of index bloat and the recovery enabled by automatic compaction. Using a write-intensive OLTP simulation with 30 concurrent threads:

| Measurement | Before Workload | After Workload | After Compaction |
|---|---|---|---|
| Logical reads (per 1,000 row query) | 25 | 1,610 | 35 |
| Page density | 99.51% | 52.71% | 96.11% |
| Total pages in index | 962 | 4,394 | 1,065 |

After running for only a few minutes following workload completion, automatic compaction reduced logical reads by approximately **98%** - returning performance nearly to the pre-fragmentation baseline. In a continuous workload scenario, compaction runs continuously alongside the workload, preventing fragmentation from accumulating in the first place.

Fewer logical reads directly translates to lower CPU usage, reduced memory pressure on the buffer pool, and less disk I/O - all of which affect your Azure SQL tier sizing and your cloud bill.

## How It Differs from Traditional Index Maintenance

Understanding the architectural difference helps explain why this matters operationally. Traditional index rebuild operations:

- Process **every page** in an index or partition, regardless of whether it was recently modified
- Require substantial free space in data files (often equal to the size of the index being rebuilt)
- Are resource-intensive enough to require off-hours scheduling
- Provide a point-in-time fix that begins degrading immediately as the workload resumes

Automatic index compaction:

- Acts **only on recently modified pages**, dramatically reducing overhead
- Requires no pre-allocated free space
- Runs continuously as a low-priority background process
- Prioritizes workload concurrency - pages with active locks are skipped and revisited later
- Maintains higher average page density **over time**, not just immediately after a maintenance window

One important callout from the [official documentation](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/automatic-index-compaction): if your existing indexes already have low page density (from years of maintenance neglect), it is worth running a one-time index reorganize before enabling compaction. That establishes a compact baseline; from there, automatic compaction maintains it.

## Broader Context: The SQL Server 2025 Engine Improvements

Automatic Index Compaction arrives on top of a foundation of engine-level improvements that have been rolling out since mid-2025. For Azure SQL Managed Instance customers, the [SQL Server 2025 update policy reached general availability in March 2026](https://techcommunity.microsoft.com/blog/azuresqlblog/ga-of-update-policy-sql-server-2025-for-azure-sql-managed-instance/4498802), delivering a fixed-version SQL engine that matches SQL Server 2025 while retaining database portability for on-premises deployments.

Two features available under this policy are particularly relevant to performance and resource consumption:

**Optimized Locking** - Now [generally available and enabled by default](https://learn.microsoft.com/en-us/sql/relational-databases/performance/optimized-locking) in Azure SQL Database and Azure SQL Managed Instance (with Always-up-to-date and SQL Server 2025 policies). Optimized locking reduces the number of row locks held for the duration of a transaction through two mechanisms: Transaction ID (TID) locking and Lock After Qualification (LAQ). In practical terms, updating 1,000 rows previously required holding 1,000 exclusive locks until transaction commit; with optimized locking, those row locks are released immediately and only a single TID lock is held. This reduces lock memory consumption significantly and dramatically reduces the risk of lock escalation - a common cause of performance degradation under concurrent write workloads.

**Degrees of Parallelism (DOP) Feedback** - [GA since July 2025](https://learn.microsoft.com/en-us/sql/relational-databases/performance/intelligent-query-processing-degree-parallelism-feedback), this Intelligent Query Processing feature automatically tunes the parallelism used for repeating queries based on observed elapsed time and wait statistics. Queries that were previously over-parallelized (consuming more CPU threads than necessary) are adjusted automatically - freeing those threads for other work.

For Azure SQL Managed Instance customers still on the SQL Server 2022 update policy, now is the time to plan a migration to the SQL Server 2025 or Always-up-to-date policy. The SQL Server 2022 policy reaches end of mainstream support on January 11, 2028, after which instances will be automatically upgraded.

## Why This Matters for Government

Government database workloads have several characteristics that make these optimizations especially impactful:

**Fiscal stewardship.** Every dollar saved on Azure infrastructure is a dollar that stays in the public budget. Storage consumption directly drives costs in Azure SQL - both in terms of database tier sizing and backup storage. Automatic Index Compaction can meaningfully reduce the storage footprint of heavily written databases - permit systems, case management platforms, 311 service records, licensing databases - without any application refactoring budget. Reduced I/O consumption can also allow workloads to operate effectively at a lower service tier.

**Lean IT operations.** Many state and local government IT departments operate with limited DBA staff relative to their database portfolio. Eliminating the need to author, schedule, monitor, and troubleshoot index maintenance jobs reduces operational burden and removes a common source of after-hours incidents (maintenance jobs running long, conflicting with backups, or degrading performance during business hours when they run late).

**Compliance and data retention.** A smaller, more compact database is easier to manage under records retention policies. Reduced storage footprint simplifies capacity planning for agencies operating under tight procurement cycles or storage quotas. Time-based immutable Long-Term Retention backups - [now generally available for Azure SQL Database](https://learn.microsoft.com/en-us/azure/azure-sql/database/backup-immutability) - pair well with a compact database footprint for audit-defensible archival. (Note: legal hold immutability remains in preview.)

**Citizen-facing performance.** Agencies running high-volume transactional workloads - motor vehicle systems, public assistance portals, tax payment processing - benefit directly from the I/O and memory improvements that compaction delivers. Fewer logical reads means faster query response times under concurrent load, improving the experience for residents interacting with government services.

**Security posture.** The [Versionless Keys for Transparent Data Encryption](https://techcommunity.microsoft.com/blog/azuresqlblog/versionless-keys-for-transparent-data-encryption-in-azure-sql-database-generally/4502969), also reaching GA in March 2026, simplifies key rotation for databases protected with customer-managed keys in Azure Key Vault. Rather than manually updating TDE configuration to reference a new key version after each rotation, versionless keys always reference the latest version automatically. For agencies with key rotation mandates under FedRAMP or NIST SP 800-53 controls, this removes a manual operational step that was a common source of misconfiguration risk.

## Availability Considerations: Azure Government vs. Commercial

Azure SQL Database and Azure SQL Managed Instance are available in Azure Government regions and are authorized at [FedRAMP High](https://learn.microsoft.com/en-us/azure/azure-government/compliance/azure-services-in-fedramp-auditscope). However, preview features typically reach Azure Government after their commercial availability, and availability timelines can vary.

Agencies operating in Azure Government should:

- Verify Automatic Index Compaction preview availability in their specific region through the [Azure Government feature availability documentation](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) or by contacting their Microsoft account team
- Note that the SQL Server 2025 update policy for SQL Managed Instance is GA in commercial and should follow in Azure Government per standard promotion timelines
- Plan for testing preview features in non-production environments before broad adoption

For organizations in M365 GCC tenants, it is worth noting that SQL database in Microsoft Fabric - which also supports Automatic Index Compaction in preview - is subject to Fabric's own GCC availability timeline. Azure SQL Database and Azure SQL Managed Instance in Azure commercial or Azure Government remain the recommended paths for database workloads in GCC environments today.

## Getting Started

To enable Automatic Index Compaction on an existing Azure SQL Database, or on an Azure SQL Managed Instance running the Always-up-to-date update policy:

1. Optionally, run a one-time index reorganize on heavily fragmented indexes to establish a compact baseline:

```sql
-- Check fragmentation first
SELECT OBJECT_NAME(ips.object_id) AS table_name,
       i.name AS index_name,
       ips.avg_fragmentation_in_percent,
       ips.page_count
FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED') ips
JOIN sys.indexes i ON ips.object_id = i.object_id AND ips.index_id = i.index_id
WHERE ips.avg_fragmentation_in_percent > 30
  AND ips.page_count > 1000
ORDER BY ips.avg_fragmentation_in_percent DESC;
```

2. Enable automatic index compaction:

```sql
ALTER DATABASE [your-database-name] SET AUTOMATIC_INDEX_COMPACTION = ON;
```

3. Monitor compaction activity using the system catalog:

```sql
SELECT database_id, name, is_automatic_index_compaction_on
FROM sys.databases
WHERE name = DB_NAME();
```

For deeper monitoring of compaction statistics, Microsoft's documentation describes using [extended events to observe the compaction process](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/automatic-index-compaction) in detail.

## Resources

- [Automatic Index Compaction - Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/indexes/automatic-index-compaction)
- [Introducing Auto Index Compaction - Azure SQL Blog](https://techcommunity.microsoft.com/blog/azuresqlblog/stop-defragmenting-and-start-living-introducing-auto-index-compaction/4500089)
- [Optimized Locking - Microsoft Learn](https://learn.microsoft.com/en-us/sql/relational-databases/performance/optimized-locking)
- [SQL Server 2025 Update Policy for Azure SQL MI - GA Announcement](https://techcommunity.microsoft.com/blog/azuresqlblog/ga-of-update-policy-sql-server-2025-for-azure-sql-managed-instance/4498802)
- [Azure SQL Database - What's New](https://learn.microsoft.com/en-us/azure/azure-sql/database/doc-changes-updates-release-notes-whats-new)
- [Azure SQL Managed Instance - What's New](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/doc-changes-updates-release-notes-whats-new)
- [Azure SQL Database Backup Immutability](https://learn.microsoft.com/en-us/azure/azure-sql/database/backup-immutability)
- [Azure Government Services Availability](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure)

