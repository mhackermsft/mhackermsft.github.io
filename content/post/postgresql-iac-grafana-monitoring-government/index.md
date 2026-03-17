---
title: 'IaC and Native Grafana Monitoring for Azure Database for PostgreSQL: What Government IT Teams Need to Know'
date: 2026-03-13T12:38:49+00:00
author: Mike Hacker
tags:
- Data Platform
- App Modernization
- Announcements
- How To
categories:
- Data Platform
summary: Azure Database for PostgreSQL now supports Terraform-based Elastic Cluster provisioning and native Grafana dashboards built into the Azure Portal, giving government IT teams powerful tools for compliant, auditable database operations.
draft: false
image_prompt: A dark blue and teal server infrastructure visualization with glowing interconnected nodes and geometric grid patterns, overlapping translucent panels showing line charts and gauge dials in green and orange hues, abstract cloud shapes flowing into a layered database cylinder icon, with soft radial gradients and a calm professional atmosphere
image: cover.jpg
---

Government IT teams are under increasing pressure to demonstrate that every infrastructure change is deliberate, documented, and repeatable. Whether it is a state agency modernizing a legacy case management system or a county government migrating citizen-facing services to the cloud, the database layer is often the most sensitive and scrutinized part of the stack. Manual deployments, ad-hoc configuration changes, and siloed monitoring tools are liabilities, not just operationally but from a compliance and audit standpoint.

Two recent Generally Available releases from Microsoft directly address these pain points for teams running [Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/azure/postgresql/): Terraform support for Elastic Clusters and native Grafana dashboards embedded directly inside the Azure Portal. Together, these capabilities form the backbone of a modern, auditable database operations practice.

---

## Infrastructure-as-Code for Azure PostgreSQL: From Configuration Drift to Declarative Control

Infrastructure-as-Code (IaC) is the practice of defining and managing infrastructure through machine-readable configuration files rather than manual portal clicks or one-off scripts. For government organizations, IaC is not just a DevOps best practice, it is rapidly becoming a compliance expectation. Frameworks like FedRAMP, NIST SP 800-53, and CISA's Secure Cloud Business Applications (SCuBA) guidance all point toward automated, version-controlled configuration management as a pillar of secure cloud operations.

Azure Database for PostgreSQL supports multiple IaC approaches, each suited to different team maturity levels and tooling ecosystems.

### Bicep: Azure-Native IaC

[Bicep](https://learn.microsoft.com/en-us/azure/azure-resource-manager/bicep/overview) is Microsoft's domain-specific language for deploying Azure resources declaratively. It compiles down to ARM (Azure Resource Manager) templates and integrates seamlessly with Azure DevOps and GitHub Actions pipelines. For government teams already operating within the Azure ecosystem, Bicep offers the most native experience.

A minimal Bicep file to deploy a zone-redundant Azure PostgreSQL Flexible Server looks like this:

```bicep
resource pgServer 'Microsoft.DBforPostgreSQL/flexibleServers@2021-06-01' = {
  name: serverName
  location: location
  sku: {
    name: 'Standard_D4ds_v5'
    tier: 'GeneralPurpose'
  }
  properties: {
    version: '16'
    administratorLogin: administratorLogin
    administratorLoginPassword: administratorLoginPassword
    highAvailability: {
      mode: 'ZoneRedundant'
    }
    storage: {
      storageSizeGB: 256
    }
    backup: {
      backupRetentionDays: 14
      geoRedundantBackup: 'Enabled'
    }
  }
}
```

Every parameter is version-controlled. Every deployment is repeatable. Reviewers and auditors can see exactly what was deployed, when, and by whom - directly from a Git commit history.

See the full [Azure PostgreSQL Bicep quickstart](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/quickstart-create-server-bicep) on Microsoft Learn.

### Terraform: Cross-Cloud IaC with Elastic Cluster Support Now GA

For organizations using Terraform, particularly those managing infrastructure across Azure Commercial and Azure US Government environments, the [AzureRM Terraform provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs) has long supported Azure Database for PostgreSQL Flexible Server. As of February 2026, Microsoft announced that **Terraform support for Elastic Clusters is now Generally Available**.

Azure Database for PostgreSQL Elastic Clusters enable horizontal scaling across multiple nodes for large or fast-growing workloads. With the GA of Terraform support, these clusters can now be fully provisioned and managed through code:

```hcl
resource "azurerm_postgresql_flexible_server" "elastic_cluster" {
  name                   = "pg-elastic-cluster"
  resource_group_name    = var.resource_group_name
  location               = var.location
  administrator_login    = var.admin_username
  administrator_password = var.admin_password
  version                = "17"
  sku_name               = "GP_Standard_D4ds_v5"
  storage_mb             = 131072

  cluster {
    size = 3
  }
}
```

This means government teams can now bring elastic, horizontally-scalable PostgreSQL deployments fully into their Terraform workflows, automating environment replication, integrating into CI/CD pipelines, and enforcing infrastructure standards across dev, staging, and production environments.

Learn more about [Azure Database for PostgreSQL Elastic Clusters](https://learn.microsoft.com/en-us/azure/postgresql/elastic-clusters/concepts-elastic-clusters).

### ARM Templates: The Foundation

For teams not yet using Bicep or Terraform, Azure Resource Manager (ARM) templates provide the baseline IaC capability. ARM templates are JSON-based, natively supported by Azure, and can be deployed via Azure CLI, PowerShell, or Azure DevOps. Every Bicep file compiles to ARM, so both approaches are interchangeable at deployment time.

The key benefit of any of these approaches - Bicep, Terraform, or ARM - is the same: **your database infrastructure becomes a reviewable, auditable artifact in source control**, not a series of undocumented portal clicks.

---

## Native Grafana Dashboards in Azure Portal: Zero-Setup Observability

Monitoring is the other side of the compliance coin. Knowing what your database is doing, right now and historically, is essential for incident response, capacity planning, and demonstrating operational control to auditors.

As of February 2026, **native Grafana dashboards are Generally Available** directly inside the Azure Portal for Azure Database for PostgreSQL. This is a significant shift from the previous model, where teams either relied on Azure Monitor metrics charts or had to deploy and manage a separate [Azure Managed Grafana](https://azure.microsoft.com/en-us/products/managed-grafana) instance.

The new built-in experience, called [Azure Monitor Dashboards with Grafana](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/visualize-grafana-overview), provides:

- **Prebuilt PostgreSQL dashboards** accessible with no configuration - just navigate to your PostgreSQL server in the Azure Portal and select "Dashboards with Grafana"
- **Real-time visibility** into CPU, memory, storage, disk I/O, network throughput, active connections, transaction rates, and server availability
- **Log and metric correlation** - view CPU spikes alongside the exact slow query logs that caused them, aligned by timestamp
- **Azure RBAC integration** - dashboard access is governed by the same role-based access control used across all Azure resources
- **No additional cost** - the built-in dashboards are included at no extra charge
- **ARM-exportable dashboards** - dashboards are Azure resources that can be exported and deployed consistently across environments

To enable log correlation in the dashboards, you need to configure diagnostic settings to route PostgreSQL server logs to Azure Monitor Logs. This is a one-time setup step:

1. Navigate to your PostgreSQL server in the Azure Portal
2. Select **Monitoring** then **Diagnostic settings**
3. Enable log forwarding to a Log Analytics workspace
4. Return to **Dashboards with Grafana** to see logs and metrics side by side

For more detail, see the [PostgreSQL monitoring documentation](https://learn.microsoft.com/en-us/azure/postgresql/monitor/concepts-monitoring) and the [Grafana dashboards announcement blog](https://techcommunity.microsoft.com/blog/adforpostgresql/dashboards-with-grafana---now-in-azure-portal-for-postgresql/4497607).

### When to Use Azure Managed Grafana Instead

The built-in portal dashboards cover the majority of day-to-day monitoring scenarios. However, teams with more advanced needs may still want [Azure Managed Grafana](https://azure.microsoft.com/en-us/products/managed-grafana), which offers:

- Extended Grafana plugin support including community plugins
- Multi-cloud or hybrid data source connectivity
- Fine-grained multi-tenant access control
- Advanced authentication and provisioning APIs

For most government agency database teams, the native portal dashboards provide more than enough visibility without additional operational overhead.

---

## Why This Matters for Government

Government IT organizations operate under compliance frameworks that demand two things above all else: **auditability** and **consistency**. Infrastructure-as-Code and native monitoring directly serve both of these requirements in ways that manual operations simply cannot match.

**Auditability through IaC:** When a Bicep file or Terraform configuration is stored in a version-controlled repository and deployed through a CI/CD pipeline, every change to the database infrastructure is tied to an approver, a timestamp, and a code review. This is exactly the kind of change control evidence that auditors and inspectors general look for. Manual portal-based deployments, by contrast, generate sparse audit logs and rely on individual memory to reconstruct what was done and why.

**Consistency across environments:** Government agencies frequently run parallel environments for development, testing, staging, and production, sometimes across both Azure Commercial and Azure US Government. IaC makes it straightforward to ensure that each environment is configured identically, with the same security controls, backup policies, and high availability settings. Configuration drift, where production quietly diverges from what was reviewed and approved, becomes much harder when the authoritative configuration lives in code.

**Faster incident response with Grafana:** When a citizen-facing application slows down or a database becomes unresponsive, the clock is ticking. The native Grafana dashboards allow an on-call administrator to open one portal blade and immediately see whether the issue is CPU saturation, connection exhaustion, storage pressure, or a slow query - without switching tools or waiting for log exports. The ability to correlate metrics with log entries at the same timestamp cuts diagnostic time dramatically.

**Security and access control alignment:** Because the built-in Grafana dashboards are governed by Azure RBAC, they fit naturally into the access control models government teams have already established. A database administrator can see all performance metrics. A read-only auditor role can view dashboards without any risk of modifying configurations. There is no separate user management system to maintain.

**Cost and operational efficiency:** Every hour a database engineer spends managing a separate monitoring server is an hour not spent on mission-critical work. The zero-cost, zero-configuration native Grafana dashboards reduce the operational overhead of maintaining observability tooling, which is especially meaningful for lean IT teams common in city and county government environments.

> **A note on Azure Government availability:** Feature releases on Azure Commercial often precede availability on Azure Government clouds. Before deploying new capabilities, verify current feature availability for [Azure Government](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) and review the [Azure Government services availability list](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/?regions=usgov-non-regional,us-dod-central,us-dod-east,usgov-arizona,usgov-texas,usgov-virginia). The IaC deployment capabilities (Bicep, Terraform, ARM) work across both commercial and government Azure environments. The native portal Grafana dashboards should be validated for availability in your specific Azure Government region.

---

## Getting Started

If your organization is ready to adopt IaC practices for Azure PostgreSQL and take advantage of native monitoring, here is a practical path forward:

1. **Choose your IaC tool:** Bicep is the best choice for Azure-native teams. Terraform is the right choice if you manage multi-cloud or cross-region infrastructure and want a unified workflow. Both are fully supported.

2. **Start with a single environment:** Port an existing non-production PostgreSQL Flexible Server configuration into a Bicep or Terraform file. Validate the output matches your current configuration before expanding to production.

3. **Integrate with your pipeline:** Connect your IaC files to an Azure DevOps or GitHub Actions pipeline with a pull request review gate. This creates the change control workflow auditors expect.

4. **Enable diagnostic settings:** Ensure all PostgreSQL servers have diagnostic settings configured to forward logs to a Log Analytics workspace. This is the prerequisite for full log-and-metric correlation in Grafana.

5. **Explore the Grafana dashboards:** Navigate to any Azure PostgreSQL server in the Azure Portal and open **Dashboards with Grafana** to see what is available out of the box. Customize copies of the prebuilt dashboards for your specific workload needs.

---

## Resources

- [Azure Database for PostgreSQL - Overview](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/overview)
- [Quickstart: Create Azure PostgreSQL Flexible Server with Bicep](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/quickstart-create-server-bicep)
- [Azure Database for PostgreSQL Elastic Clusters](https://learn.microsoft.com/en-us/azure/postgresql/elastic-clusters/concepts-elastic-clusters)
- [February 2026 Recap: Azure Database for PostgreSQL](https://techcommunity.microsoft.com/blog/adforpostgresql/february-2026-recap-azure-database-for-postgresql/4501093)
- [Dashboards with Grafana - Now in Azure Portal for PostgreSQL](https://techcommunity.microsoft.com/blog/adforpostgresql/dashboards-with-grafana---now-in-azure-portal-for-postgresql/4497607)
- [Azure Monitor Dashboards with Grafana - Overview](https://learn.microsoft.com/en-us/azure/azure-monitor/visualize/visualize-grafana-overview)
- [Azure Database for PostgreSQL Monitoring Concepts](https://learn.microsoft.com/en-us/azure/postgresql/monitor/concepts-monitoring)
- [Configure and Access PostgreSQL Server Logs](https://learn.microsoft.com/en-us/azure/postgresql/monitor/how-to-configure-and-access-logs)
- [Azure Managed Grafana](https://azure.microsoft.com/en-us/products/managed-grafana)
- [Azure Government Services Availability](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/?regions=usgov-non-regional,us-dod-central,us-dod-east,usgov-arizona,usgov-texas,usgov-virginia)
