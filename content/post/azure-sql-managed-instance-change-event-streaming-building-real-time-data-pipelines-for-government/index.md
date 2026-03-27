---
title: 'Azure SQL Managed Instance Change Event Streaming: Building Real-Time Data Pipelines for Government'
date: 2026-03-27T12:23:21+00:00
author: Mike Hacker
tags:
- Data Platform
- App Modernization
- How To
- Announcements
categories:
- Data Platform
summary: Azure SQL Managed Instance now supports Change Event Streaming in public preview, enabling government organizations to publish real-time row-level data changes directly to Azure Event Hubs for modern event-driven architectures.
draft: false
image_prompt: A modern data pipeline visualization showing a government database streaming real-time change events through Azure Event Hubs to multiple downstream systems, with clean geometric design elements in blue and white tones suggesting reliability and security
audio: audio.mp3
image: cover.jpg
---

Government organizations run on data, and that data lives overwhelmingly in relational databases. When a permit status changes, a benefits record is updated, or a new inspection is filed, downstream systems often need to know about it immediately. Historically, achieving that kind of near real-time data flow from SQL databases required complex ETL pipelines, polling-based integrations, or third-party change data capture tools that added cost and operational burden.

Microsoft recently announced the **public preview of Change Event Streaming (CES)** for Azure SQL Managed Instance, Azure SQL Database, and SQL Server 2025. This new capability streams row-level data changes directly from your SQL database into [Azure Event Hubs](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-about), enabling modern event-driven architectures with minimal overhead and native T-SQL configuration.

## What Is Change Event Streaming?

Change Event Streaming (CES) is a built-in data integration capability that captures incremental DML changes (inserts, updates, and deletes) from tracked tables and publishes them to Azure Event Hubs in near real-time. Each change is packaged as a [CloudEvent](https://cloudevents.io/) - an industry-standard envelope format - serialized as either native JSON or Avro Binary.

Unlike traditional Change Data Capture (CDC), which writes change records back into the database, CES streams changes directly out to Azure Event Hubs. This design means **lower performance overhead** on the source database and a cleaner separation between your transactional workload and your data integration layer.

The events include rich metadata: the source database, schema, table name, column definitions, primary key values, and both the previous and current row values. This gives downstream consumers everything they need to understand and act on each change without querying back to the source.

For complete details, see the official [Change Event Streaming overview](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/overview?view=sql-server-ver17) documentation on Microsoft Learn.

## Key Features and Capabilities

### Native T-SQL Configuration

CES is configured entirely through T-SQL stored procedures, which means your database administrators can set it up without learning new tools or deploying additional infrastructure. The core workflow is straightforward:

1. Enable event streaming on the database with `sys.sp_enable_event_stream`
2. Create a streaming group that defines the destination, authentication, message size, and partitioning with `sys.sp_create_event_stream_group`
3. Add tables to the streaming group with `sys.sp_add_object_to_event_stream_group`

Here is a simplified example of what the configuration looks like:

```sql
-- Enable event streaming for the database
EXEC sys.sp_enable_event_stream;

-- Create the streaming group targeting Azure Event Hubs
EXEC sys.sp_create_event_stream_group
    @stream_group_name    = N'PermitChanges',
    @destination_type     = N'AzureEventHubsAmqp',
    @destination_location = N'mynamespace.servicebus.windows.net/myeventhub',
    @destination_credential = MyCredential,
    @max_message_size_kb  = 256,
    @partition_key_scheme  = N'Table';

-- Add a table to the streaming group
EXEC sys.sp_add_object_to_event_stream_group
    N'PermitChanges',
    N'dbo.BuildingPermits';
```

For complete configuration instructions, see the [Configure Change Event Streaming](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/configure?view=sql-server-ver17) guide.

### Protocol Flexibility

CES supports both the **AMQP protocol** (the native Azure Event Hubs protocol) and the **Apache Kafka protocol**. This flexibility means you can integrate with existing Kafka-based ecosystems without deploying separate Kafka clusters. Azure Event Hubs provides [built-in Kafka compatibility](https://learn.microsoft.com/en-us/azure/event-hubs/azure-event-hubs-apache-kafka-overview), so teams already invested in Kafka tooling can consume CES events through familiar client libraries.

### Secure Authentication Options

For Azure SQL Managed Instance, CES supports **Microsoft Entra authentication** (the most secure method) as well as Shared Access Signature (SAS) token authentication. Microsoft Entra authentication aligns with government zero-trust security models and eliminates the need to manage shared keys or tokens manually.

### Rich Event Payload

Each CES event follows the [CloudEvents specification](https://github.com/cloudevents/spec) and includes:

- **Operation type**: Whether the change was an INSERT, UPDATE, or DELETE
- **Timestamp**: UTC timestamp of when the commit occurred
- **Schema metadata**: Database name, schema, table, and full column definitions
- **Primary key values**: For identifying the specific row that changed
- **Old and current values**: The row state before and after the change

This self-describing format means consumers do not need external schema registries or separate metadata lookups to process events. For the full message format specification, see the [CES message format documentation](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/message-format?view=sql-server-ver17).

### Scale and Monitoring

CES supports up to **4,096 streaming groups** with up to **40,000 tables per group**, which provides ample headroom for even the largest government database environments. Monitoring is available through system DMVs including `sys.dm_change_feed_errors` for delivery errors and `sys.dm_change_feed_log_scan_sessions` for log scan activity.

## Practical Use Cases for Government

The value of real-time data streaming from relational databases becomes clear when you consider the types of systems government organizations operate:

### Cross-System Data Synchronization

Government agencies often maintain multiple systems that need consistent views of the same data: a permitting system, a GIS platform, a citizen portal, and an analytics warehouse, for example. With CES, changes in the authoritative SQL database are automatically streamed to Event Hubs, where each downstream system can consume updates independently. This eliminates fragile point-to-point integrations and nightly batch syncs.

### Real-Time Operational Dashboards

When a city needs to track permit processing times, benefits claim volumes, or inspection completions in real time, CES provides the foundation. Events from SQL Managed Instance flow into Event Hubs and can be consumed by [Azure Stream Analytics](https://learn.microsoft.com/en-us/azure/stream-analytics/stream-analytics-introduction), Azure Functions, or any Kafka-compatible consumer to power live dashboards and alerts.

### Audit and Compliance Tracking

CES events include both the old and new values for every changed row, creating a natural audit trail. Government organizations that need to track who changed what (and when) in sensitive databases can route these events to long-term storage for compliance and forensic analysis. Combined with [Event Hubs Capture](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-capture-overview), events can be automatically written to Azure Blob Storage or Azure Data Lake for retention.

### Event-Driven Application Architectures

Modern government applications increasingly follow event-driven patterns, where services react to changes rather than polling for them. CES enables this pattern directly from the database layer. When a record changes, downstream microservices receive the event through Event Hubs and can trigger workflows, notifications, or secondary processing without any coupling to the source database.

## Important Considerations

Before adopting CES, government IT teams should be aware of several key points:

- **Preview status**: CES is currently in public preview. Production workloads should be evaluated carefully during this phase. Review Microsoft's [preview supplemental terms](https://azure.microsoft.com/support/legal/preview-supplemental-terms/) for details.
- **Update policy requirement**: Azure SQL Managed Instance must be configured with the **SQL Server 2025** or **Always-up-to-date** [update policy](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/update-policy) to use CES.
- **Full recovery model required**: The source database must use the full recovery model since CES relies on reading the transaction log.
- **Mutual exclusivity with CDC**: CES cannot be enabled on a database that already uses Change Data Capture or transactional replication. Organizations currently using CDC will need to plan a migration path.
- **No initial snapshot**: CES only streams changes that occur after it is enabled. Existing data must be loaded through other means if a full initial sync is needed.
- **Azure Government availability**: Both Azure SQL Managed Instance and Azure Event Hubs are [available in Azure Government regions](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure). However, preview features may have a different availability timeline in Azure Government compared to Azure commercial. Check the [Azure Government services comparison](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) page for the latest status before planning deployments in government cloud environments.

For a comprehensive list of frequently asked questions, including performance impact, schema change handling, and high availability considerations, see the [CES FAQ](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/frequently-asked-questions-faq?view=sql-server-ver17).

## Why This Matters for Government

Government IT organizations face a unique set of pressures: aging infrastructure, growing citizen expectations for digital services, strict compliance mandates, and the need to integrate data across siloed departments. Change Event Streaming addresses several of these challenges directly.

**Reduced integration complexity.** Instead of building and maintaining custom ETL jobs, polling scripts, or trigger-based replication, CES provides a built-in, declarative way to stream database changes. This is especially valuable for smaller government IT teams that need to do more with fewer resources.

**Lower total cost of ownership.** By eliminating the need for third-party CDC tools or middleware, CES reduces both licensing costs and operational overhead. The feature is included with Azure SQL Managed Instance, and [Azure Event Hubs](https://azure.microsoft.com/pricing/details/event-hubs/) consumption costs are straightforward and predictable.

**Stronger compliance posture.** The ability to capture every row-level change with both old and new values creates a built-in audit capability that supports regulatory requirements common in government, from financial auditing to data governance mandates.

**Foundation for modernization.** Many government agencies are in the middle of multi-year modernization journeys, moving from monolithic applications to microservices, from batch processing to real-time analytics, and from on-premises infrastructure to cloud-native architectures. CES provides a critical building block: the ability to make relational database changes available to the rest of the technology ecosystem in real time, without modifying existing applications.

**Security alignment.** With Microsoft Entra authentication support, CES fits naturally into zero-trust security models. There are no shared credentials to rotate, and access control integrates with existing identity governance.

## Getting Started

To begin exploring Change Event Streaming, government IT teams should:

1. **Review the documentation**: Start with the [CES overview](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/overview?view=sql-server-ver17) and [configuration guide](https://learn.microsoft.com/en-us/sql/relational-databases/track-changes/change-event-streaming/configure?view=sql-server-ver17) on Microsoft Learn.
2. **Set up a test environment**: Use the [free Azure SQL Managed Instance](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/free-offer?view=azuresql) offer (available for the first 12 months) to experiment with CES without incurring compute costs.
3. **Provision Azure Event Hubs**: Create an [Event Hubs namespace](https://learn.microsoft.com/en-us/azure/event-hubs/event-hubs-create) to receive events.
4. **Identify a pilot use case**: Choose a non-critical database table with moderate change volume to test the end-to-end flow before expanding to production scenarios.
5. **Validate in your target cloud**: If your organization operates in Azure Government, confirm CES preview availability for your region before planning production timelines.

Change Event Streaming represents a meaningful step forward in how SQL Server databases can participate in modern, event-driven architectures. For government organizations that depend on relational databases as their system of record, this capability opens the door to real-time data integration patterns that were previously complex and costly to implement.
