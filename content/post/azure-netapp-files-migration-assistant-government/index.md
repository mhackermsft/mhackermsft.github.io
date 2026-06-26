---
title: 'Lifting Legacy NAS to the Cloud: A Hands-On Look at the Azure NetApp Files Migration Assistant'
date: 2026-06-26T14:05:30+00:00
author: Mike Hacker
tags:
- App Modernization
- How To
- Announcements
categories:
- Cloud Migration
summary: The Azure NetApp Files migration assistant portal experience is generally available in the June 2026 Azure NetApp Files update, using ONTAP replication for low-downtime migration from on-premises storage or Cloud Volumes ONTAP to Azure NetApp Files, with Azure Government feature availability checked against current documentation.
draft: false
image_prompt: Azure Government cloud migration illustration showing legacy NAS storage replicating securely into Azure NetApp Files through private connectivity, with government IT operators reviewing a controlled cutover plan.
image: cover.png
audio: audio.mp3
---

Most government IT estates still run on a foundation of file-based storage: department home directories, GIS datasets, permit and case management applications, engineering CAD repositories, and the occasional EDA or HPC workload in a research or public-safety lab. These systems often live on aging on-premises NAS appliances that are expensive to refresh, hard to protect, and increasingly difficult to staff. Moving them to the cloud has historically meant a risky file-copy project with robocopy or rsync, long downtime windows, and permissions that must be revalidated after the move.

In the June 2026 update to Azure NetApp Files, Microsoft announced that the Azure NetApp Files migration assistant portal experience is generally available. The feature is designed to accelerate migrations of business-critical applications and data from on-premises storage or Cloud Volumes ONTAP into Azure NetApp Files by using ONTAP's built-in replication engine. See the [What's new in Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/whats-new) page for the official GA note.

## What the migration assistant does

Azure NetApp Files is an Azure-native, first-party, enterprise file storage service. Microsoft describes it as running on a fault-tolerant bare-metal fleet powered by ONTAP, with support for SMB, NFS, dual-protocol volumes, data protection, and multiple performance tiers. That matters for migration because many agencies already operate NetApp ONTAP environments on-premises.

Per the GA announcement, the migration assistant provides three practical advantages over a file-copy-only approach:

- **Storage-efficient transfer.** It uses ONTAP replication for baseline and incremental updates, reducing the amount of data that must traverse private connectivity during the migration.
- **Lower cutover risk.** Incremental updates keep the destination closer to the source before the maintenance window, so the final update can be much smaller than a full recopy.
- **Data fidelity.** Volume migration includes source volume snapshots for primary data protection and maintains directory and file metadata, including security attributes.

That last point is easy to underestimate. When permissions do not survive a file migration, the technical cutover becomes a help-desk and compliance problem. Preserving security metadata is a core reason to evaluate the migration assistant instead of treating file migration as a simple copy job.

## How the replication topology fits together

Conceptually, the migration flow is an asynchronous replication relationship:

1. **Source.** An on-premises NetApp ONTAP system or Cloud Volumes ONTAP instance hosts the volume you want to move.
2. **Replication transport.** ONTAP replication sends an initial baseline and then incremental changes. Private connectivity, such as ExpressRoute or site-to-site VPN, carries migration traffic into the Azure virtual network.
3. **Destination.** Azure NetApp Files provides the target volume in a capacity pool and presents it through a delegated subnet in the virtual network.
4. **Cutover.** During the maintenance window, you quiesce writes, run the final incremental update, break the relationship, and repoint clients to the Azure NetApp Files endpoint.

After cutover, the destination is a normal Azure NetApp Files volume. Native capabilities such as snapshots, backup, cross-zone replication, cross-region replication, Active Directory integration, LDAP-backed NFS access patterns, and advanced ransomware protection can be considered as part of the day-two operating model.

## Sizing the landing zone: capacity pools and service levels

The most important Azure-side design choice is the capacity pool, because service level and QoS model determine the performance envelope available to migrated volumes. Current Azure NetApp Files documentation lists five service levels: Elastic, Flexible, Standard, Premium, and Ultra.

| Service level | Throughput model | Typical workload fit |
|---|---|---|
| Standard | Up to 16 MiB/s per 1 TiB | Home directories, general file shares, archival |
| Premium | Up to 64 MiB/s per 1 TiB | Line-of-business apps and departmental shares |
| Ultra | Up to 128 MiB/s per 1 TiB | Databases, EDA/HPC, and latency-sensitive apps |
| Flexible | Capacity and throughput tuned independently in a manual QoS pool | Oracle, SAP HANA, and mixed capacity/performance needs |
| Elastic (preview) | 32 MiB/s per 1 TiB with shared QoS and zone redundancy | Workloads that need in-region zone resilience where the preview is available |

Two Government-specific caveats are important. First, the [Azure NetApp Files for Azure Government](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-government) documentation states that Elastic zone-redundant storage is not available in supported Azure Government regions. Second, Microsoft lists Red Hat IdM and Oracle Unified Directory LDAP support as a preview enhancement in the May 2026 Azure NetApp Files update. Agencies should treat those directory integrations as preview unless their own subscription, region, and support guidance confirm otherwise.

A few practical sizing notes from the Azure NetApp Files limits and service-level documentation:

- For Flexible, Standard, Premium, and Ultra, a capacity pool can be as small as 1 TiB and as large as 2,048 TiB. The 1-TiB minimum applies only when all volumes in the pool use Standard network features. Pools that include Basic network features still require a 4-TiB minimum.
- Regular volumes for Flexible, Standard, Premium, and Ultra range from 50 GiB to 100 TiB. Large volumes start at 50 TiB and can scale to 1 PiB by default, with larger limits available only under documented conditions.
- Elastic has separate limits: capacity pools range from 1 TiB to 128 TiB, and volumes range from 1 GiB to 16 TiB.
- In an auto QoS pool, the volume throughput limit is based on the volume quota and service-level rate. A 2-TiB Premium volume receives a 128 MiB/s throughput limit. In a manual QoS pool, you assign volume throughput independently within the pool's total throughput budget.

A useful migration tactic is to land the workload in a higher service level during the heavy migration phase, then move the volume to an existing lower-tier capacity pool after cutover if steady-state performance allows it. The documented service-level change is an in-place volume move between capacity pools in the same NetApp account. It does not require data migration and does not affect volume access. A move from a higher service level back down to a lower service level has a documented 24-hour wait requirement after an upscale move.

## Provisioning shape with Azure CLI

For Azure Government subscriptions, set the Azure CLI cloud context to AzureUSGovernment before you create resources, then sign in with the appropriate Government tenant and subscription. Run the CLI from an environment that is configured to reach Azure Government endpoints and that meets your agency's access-control requirements.

```bash
az cloud set --name AzureUSGovernment
az login

az netappfiles account create --resource-group rg-gov-files --name netappgovacct --location usgovvirginia

az netappfiles pool create --resource-group rg-gov-files --account-name netappgovacct --name pool-premium --size 4 --service-level Premium
```

The Azure CLI reference describes `--size` for `az netappfiles pool create` as the provisioned size of the pool in tebibytes. This example uses a 4-TiB Premium pool, which remains valid for pools that must meet the 4-TiB minimum. If you plan to use the 1-TiB minimum with only Standard network features, validate the current CLI behavior, regional quota, and service limits before automating production deployment.

The destination volumes and replication relationship are created through the migration assistant portal workflow. Treat the CLI above as landing-zone scaffolding only, and validate names, region, quota, subnet, identity, and directory settings against your agency standards before production use.

## Networking and VNet integration

Azure NetApp Files volumes are provisioned into a delegated subnet in a virtual network. For migration planning, the key prerequisites are:

- Private connectivity between the source data center and the Azure virtual network, commonly ExpressRoute for sustained migration traffic or site-to-site VPN where appropriate.
- A subnet delegated to `Microsoft.NetApp/volumes` for standard Azure NetApp Files volumes. Elastic zone-redundant storage uses `Microsoft.NetApp/elasticVolumes`, but that service level is not available in Azure Government according to the current Azure NetApp Files Government documentation.
- For SMB volumes, an Active Directory connection on the NetApp account before volume creation. Azure NetApp Files supports one Active Directory connection per NetApp account as the default model.
- For NFS environments that use LDAP, confirm the supported directory type and feature status. The published LDAP with extended groups documentation covers AD DS and Microsoft Entra Domain Services, while the May 2026 update notes list Red Hat IdM and Oracle Unified Directory support as preview.

Plan DNS and client repointing before the cutover window. Whether you use DFS namespaces, a CNAME, or an automation script to remount NFS exports, the goal is to make the endpoint change controlled, documented, and reversible.

## Cutover planning

A clean cutover sequence looks like this:

1. Establish the replication relationship and let the baseline complete during a low-traffic period.
2. Let incremental updates run until the destination is closely tracking the source.
3. Schedule a maintenance window and quiesce writes on the source application.
4. Run the final incremental update.
5. Break the relationship, repoint clients or DNS, and validate access and permissions.
6. Keep the source read-only as a rollback safety net until operational confidence is high, then decommission it through normal change control.

The core benefit is not that migration becomes automatic. The benefit is that the risky work moves from a one-time full-copy event into a managed replication workflow with snapshots, metadata preservation, and a shorter final synchronization step.

## A note on Azure Government regions

Azure NetApp Files is available in Azure Government, and Microsoft maintains a dedicated [Azure NetApp Files for Azure Government](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-government) page for feature differences. That page states that all Azure NetApp Files features available in Azure public cloud are available in supported Azure Government regions except for the listed exclusions, which currently include Elastic zone-redundant storage and the preview cool-access enhancement for Premium and Ultra.

If your disaster-recovery design depends on cross-region replication after the migration, confirm the supported regional pair. The documented Azure Government cross-region replication pairs are US Gov Arizona with US Gov Texas and US Gov Virginia with US Gov Texas. Also validate service availability with the [Products available by region](https://azure.microsoft.com/explore/global-infrastructure/products-by-region/?products=netapp&rar=true&regions=all) page before committing a design, because feature availability can differ between Azure public cloud and Azure Government.

## Why This Matters for Government

File-server modernization is one of the highest-leverage cloud moves a public-sector organization can make, but it is often blocked by migration risk rather than technical capability. The migration assistant changes that risk profile in practical ways:

- **Shorter downtime supports real maintenance windows.** A permitting system or case-management share cannot be offline for days. Incremental replication can make the final cutover a planned operational event rather than a full recopy.
- **Storage-efficient transfer respects constrained budgets.** Reducing transferred data during baseline and incremental updates can reduce network transfer cost and helps agencies fit migration work into fixed budget cycles.
- **Preserved security attributes protect compliance posture.** Carrying security metadata forward reduces the chance that resident PII, regulated records, or justice and health data becomes overexposed after the move.
- **Azure Government alignment matters.** Microsoft documents Azure Government as assessed and authorized at FedRAMP High and as providing contractual commitments for customer data storage in the United States and access by screened US persons. Those commitments can be important for agencies working through FedRAMP, CJIS, IRS Pub 1075, HIPAA, or state-level control mapping.
- **Day-two operations improve.** Once the data lands in Azure NetApp Files, teams can evaluate native snapshots, backup, replication, directory integration, and advanced ransomware protection without rebuilding the storage platform around separate migration tooling.

For agencies facing another storage-array refresh, the June 2026 GA of the migration assistant is a timely reason to evaluate whether legacy NAS workloads can move to an Azure-native, ONTAP-powered file platform with less migration risk than traditional file-copy approaches.

## Next steps

- Review [What is Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-introduction) for the full service overview.
- Plan performance and capacity with the [service levels guide](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-service-levels) and [resource limits](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-resource-limits).
- Confirm Government feature availability with [Azure NetApp Files for Azure Government](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-government) and validate DR topology against the [replication documentation](https://learn.microsoft.com/en-us/azure/azure-netapp-files/cross-region-replication-introduction).
