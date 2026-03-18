---
title: 'From Legacy to Open Source Leadership: The Government Case for PostgreSQL on Azure'
date: 2026-03-18T18:17:52+00:00
author: Mike Hacker
tags:
- Data Platform
- App Modernization
- Open Source
categories:
- Azure Database for PostgreSQL
summary: Government agencies modernizing their data infrastructure can reduce licensing costs and gain vendor independence by migrating to PostgreSQL on Azure, backed by enterprise-grade security, FedRAMP High compliance, and the fully managed capabilities of Azure Database for PostgreSQL Flexible Server.
draft: false
image_prompt: A massive stone government building column cracking open to reveal a glowing PostgreSQL elephant logo emerging from inside, surrounded by rising streams of blue light and floating open-source code symbols, dramatic low-angle golden hour lighting with light rays cutting through mist. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
---

Government IT organizations have been running on Oracle Database and SQL Server for decades. These platforms built the foundation for everything from property tax systems to permitting workflows to public safety dispatch databases. But that foundation is increasingly expensive to maintain for agencies carrying the cost of on-premises infrastructure, and it can constrain the modernization roadmaps that government technology leaders need to execute.

A growing number of government agencies are making a strategic shift: moving to open-source PostgreSQL on Azure. Not because it is the cheapest option in a vacuum, but because it is the right option when total cost of ownership, vendor independence, security posture, and long-term capability are all weighed together. This post makes the case for that shift and explains what the migration journey looks like.

---

## The CxO Conversation: Why Open Source Databases Deserve Budget Attention

Oracle Database licensing has long been among the most expensive recurring costs in government IT budgets. The combination of per-processor fees, costly support agreements, and licensing audits that can result in unexpected true-up costs creates a financial environment that is both unpredictable and difficult to defend to finance committees and budget officers.

PostgreSQL changes the cost math significantly. It is released under the permissive PostgreSQL License, with no per-core fees, no support contracts required to access the software, and no audits triggered by growth in your deployment footprint. For a county agency that adds fifty additional permit-processing users, the database license cost does not change. For an agency adding a new GIS workload that needs a large-memory instance, there is no license tier to upgrade.

The financial case is not just about license elimination. It is also about staffing leverage. Because PostgreSQL is one of the world's most widely deployed open-source relational databases, the available talent pool is deep and growing. Government agencies are increasingly finding that hiring PostgreSQL-skilled database administrators is easier and faster than competing for scarce Oracle specialists.

Microsoft's commitment to PostgreSQL further reduces risk. The company has employees who actively contribute to the PostgreSQL open-source project, writing code, reviewing commits, and participating in community governance. Azure Database for PostgreSQL is not just a hosted database: it is backed by one of the largest enterprise contributors to the open-source ecosystem itself.

---

## The Migration Path: From Oracle  to PostgreSQL on Azure

The question for most government IT leaders is not whether to move, but how. The migration journey from Oracle to PostgreSQL is well-documented, with tooling and professional guidance widely available.

**From Oracle:** Microsoft provides comprehensive guidance for Oracle-to-PostgreSQL migration using [Ora2Pg](https://learn.microsoft.com/en-us/azure/postgresql/migrate/how-to-migrate-from-oracle), a free, open-source tool that connects directly to an Oracle database, scans its structure, and generates SQL scripts compatible with PostgreSQL. The tool supports:

- Automated schema conversion including tables, indexes, sequences, and views
- Assessment of PL/SQL stored procedures that require manual conversion
- Full data export in PostgreSQL-compatible format
- Estimation of migration complexity and effort before the first byte of data moves

The Ora2Pg assessment report gives IT leadership a clear picture of migration scope in advance. Objects that can be automatically converted are quantified separately from those requiring custom development, making project planning and budget estimation far more accurate than traditional migration guesswork.

**From other PostgreSQL sources:** For agencies already running PostgreSQL on-premises or in a different cloud, the [migration service in Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql) provides both online and offline migration options with no database size limits and no complex setup. Online migration keeps source databases running during the migration window, minimizing downtime for citizen-facing services.

---

## Azure Database for PostgreSQL Flexible Server: Enterprise-Ready for Government

Once migrated, [Azure Database for PostgreSQL Flexible Server](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview) provides the managed platform foundation. Key capabilities for government workloads include:

**Security and compliance:** All data at rest is encrypted using AES-256 through Azure Storage server-side encryption, covering primary servers, replicas, point-in-time recovery, and backups. Data in transit is protected with TLS, with TLS 1.2 as the enforced minimum and TLS 1.3 available for deployments requiring it. Microsoft Entra ID integration provides centralized authentication with conditional access policies and managed identities for application connections, eliminating the need for stored database credentials. Private endpoint networking ensures database traffic never traverses the public internet. Azure Government cloud regions are assessed at the FedRAMP High impact level.

**High availability:** Zone-redundant high availability provisions a warm standby server in a separate availability zone with synchronous replication, enabling zero-data-loss failover. For agencies where database downtime means citizen services go offline, this configuration provides the resilience that government service-level agreements require.

**Automated operations:** Automated backups with up to 35 days of retention, AES-256 encryption, and point-in-time restore give government agencies strong data protection with minimal operational overhead. Automated patching with customizable maintenance windows keeps databases current on security updates without surprise downtime during business hours.

**AI-ready architecture:** The service includes the Azure AI extension for PostgreSQL, enabling vector search and AI-powered query capabilities directly within the database. For government agencies exploring intelligent document processing, semantic search across case files, or AI-assisted permit review, this capability can be layered on without deploying a separate AI infrastructure tier.

**Built-in connection pooling:** The pgBouncer connection pooler is included at no additional cost, supporting up to 10,000 concurrent application connections while keeping database resource usage efficient. This is particularly important for agencies running citizen-facing web applications that experience bursty concurrent load.

---

## Why This Matters for Government

Government agencies carry unique constraints that make the PostgreSQL-on-Azure combination particularly well-suited for the next phase of database modernization.

**Budget predictability:** Open-source licensing eliminates the variable cost exposure of commercial database audits and true-up events. Fixed managed service pricing for Azure Database for PostgreSQL gives IT finance teams a cost model they can defend across multi-year budget cycles.

**Compliance alignment:** FedRAMP High authorization in Azure Government, AES-256 encryption at rest with TLS 1.3 available for data in transit, Entra ID integration, and private endpoint networking address the compliance requirements that state and local government agencies face across frameworks including CISA guidance and state-specific data protection regulations.

**Vendor independence:** PostgreSQL is governed by a community, not a single commercial vendor. An agency's investment in PostgreSQL skills, schema design, and application integration is portable across providers. That optionality has real strategic value in multi-year government IT planning where procurement conditions can change.

**Talent market dynamics:** University computer science programs increasingly teach PostgreSQL as the primary relational database. Government agencies hiring early-career database professionals will find candidates who already know the platform, reducing onboarding time and training costs.

**AI readiness:** As state and local governments begin deploying AI-assisted services, a database platform with built-in vector search and AI extension support provides a foundation that does not require a separate infrastructure investment. A permitting or licensing database can become the backend for an intelligent document search capability without leaving the PostgreSQL environment.

> **A note on availability:** Azure Database for PostgreSQL Flexible Server is available in Azure Government regions. Government agencies should verify service availability in their specific environment before planning production migrations. See the [Azure Government services availability list](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/?regions=usgov-non-regional,us-dod-central,us-dod-east,usgov-arizona,usgov-texas,usgov-virginia) for current status.

---

## Getting Started

The path to PostgreSQL on Azure typically follows three phases:

1. **Assess:** Use Ora2Pg assessment reports for Oracle workloads or database analysis tools for SQL Server to build a migration complexity map. Categorize databases by complexity and business criticality. Start with lower-complexity, non-production workloads to build team confidence and surface conversion patterns early.

2. **Migrate:** Use the migration service built into Azure Database for PostgreSQL for PostgreSQL-source workloads, or Ora2Pg for Oracle-source data movement. For SQL Server sources, plan for schema conversion and application-layer refactoring using community tools like pgloader alongside partner support. Use online migration where downtime is not acceptable for citizen-facing systems.

3. **Optimize:** Once on Azure PostgreSQL, take advantage of native capabilities including zone-redundant HA, automated backups, Entra ID authentication, and built-in connection pooling. Engage your Microsoft account team to learn about the latest capabilities and roadmap for Azure Database for PostgreSQL.

---

## Resources

- [Azure Database for PostgreSQL - Overview](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview)
- [Migrate from Oracle to Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/migrate/how-to-migrate-from-oracle)
- [Migration Service in Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/migrate/migration-service/overview-migration-service-postgresql)
- [Azure Database for PostgreSQL Security Overview](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-security)
- [Azure Database for PostgreSQL Data Encryption](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/security-data-encryption)
- [Supporting ChatGPT on PostgreSQL in Azure](https://techcommunity.microsoft.com/blog/adforpostgresql/supporting-chatgpt-on-postgresql-in-azure/4490247)
- [Azure Government Services Availability](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/?regions=usgov-non-regional,us-dod-central,us-dod-east,usgov-arizona,usgov-texas,usgov-virginia)

