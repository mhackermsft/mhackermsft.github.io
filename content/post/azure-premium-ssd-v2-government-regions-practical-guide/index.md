---
title: 'Azure Premium SSD v2 Now Available in Government Regions: A Practical Guide'
date: 2026-03-31T17:34:54+00:00
author: Mike Hacker
tags:
- Data Platform
- How To
- Announcements
categories:
- Infrastructure
summary: Azure Premium SSD v2 disk storage is now generally available in Azure Government regions, bringing independently tunable IOPS, throughput, and capacity to mission-critical government workloads at lower cost than traditional Premium SSDs.
draft: false
image_prompt: A government data center with rows of glowing blue unbranded storage arrays and server racks, representing high-performance block storage powering mission-critical public sector workloads with sub-millisecond latency. No text, no labels, no logos, no branding, no letters, no numbers, no words, no writing of any kind anywhere in the image.
image: cover.jpg
audio: audio.mp3
---

Government agencies running mission-critical workloads on Azure now have access to one of the most significant improvements in block storage performance and cost efficiency in years. Azure Premium SSD v2 is generally available in Azure Government regions, including US Gov Arizona and US Gov Virginia, giving public sector organizations a powerful new storage tier that delivers sub-millisecond latency with granular, independent control over performance and capacity.

This is not just an incremental upgrade. Premium SSD v2 fundamentally changes how agencies can approach storage provisioning: instead of being locked into fixed performance tiers, IT teams can now dial in exactly the IOPS, throughput, and capacity their workloads demand and adjust on the fly without downtime.

## What Is Azure Premium SSD v2?

Azure Premium SSD v2 is a next-generation managed disk offering designed for IO-intensive enterprise workloads. It sits between the original Premium SSD (designed for general production workloads) and Ultra Disks (designed for the most extreme performance scenarios), offering a compelling balance of performance, flexibility, and cost.

The key differentiator is **independent performance tuning**. With traditional Premium SSDs, your IOPS and throughput are determined by the disk size tier you select. If you need more IOPS, you may have to provision a larger (and more expensive) disk even if you do not need the extra capacity. Premium SSD v2 eliminates this coupling entirely.

Here is what Premium SSD v2 delivers:

- **Capacity**: 1 GiB to 64 TiB, billed per GiB with no fixed tier sizes
- **IOPS**: Up to 80,000 IOPS (scales at 500 IOPS per GiB above 6 GiB)
- **Throughput**: Up to 1,200 MB/s (scales at 0.25 MB/s per provisioned IOPS)
- **Latency**: Sub-millisecond, consistently delivered 99.9% of the time
- **Baseline performance**: Every disk receives 3,000 IOPS and 125 MB/s at no additional cost, regardless of size

For detailed specifications, see the official [Azure Managed Disk types documentation](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types#premium-ssd-v2).

## Government Region Availability

Premium SSD v2 is now available in two Azure Government regions:

| Region | Availability Zone Support |
|--------|---------------------------|
| US Gov Arizona | Regional (no Availability Zones) |
| US Gov Virginia | Three Availability Zones |

This is significant because it means agencies can deploy Premium SSD v2 disks in either of these government regions, with US Gov Virginia offering full zonal redundancy for workloads that require high availability across availability zones. In US Gov Arizona, disks are deployed at the regional level without zone assignment.

For agencies already running workloads in Azure Government, the path to adoption is straightforward: Premium SSD v2 uses the same management plane, APIs, and tooling (Azure CLI, PowerShell, Azure Portal, ARM templates) as other managed disk types. You can find a step-by-step deployment guide at [Deploy a Premium SSD v2](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-deploy-premium-v2).

## How Premium SSD v2 Compares to Premium SSD

Understanding when to use Premium SSD v2 versus the original Premium SSD is essential for making smart procurement decisions.

| Feature | Premium SSD | Premium SSD v2 |
|---------|-------------|----------------|
| Disk sizes | Fixed tiers (P1 through P80) | Any size from 1 GiB to 64 TiB |
| Max IOPS | 20,000 | 80,000 |
| Max throughput | 900 MB/s | 1,200 MB/s |
| IOPS/throughput tuning | Tied to disk size tier | Independent, adjustable without downtime |
| Host caching | Supported | Not supported |
| OS disk support | Yes | No (data disks only) |
| Latency | Single-digit millisecond | Sub-millisecond |
| Bursting | Yes (credit-based and on-demand) | Not needed due to granular provisioning |

The most impactful difference for government workloads is the decoupling of performance from capacity. Agencies frequently run databases and applications where IOPS requirements are high but storage capacity needs are modest. With Premium SSD, you might need to provision a P50 (4 TiB) disk just to get 7,500 IOPS, even if your database only uses 200 GiB of space. With Premium SSD v2, you can provision exactly 200 GiB and set your IOPS to whatever your workload requires, paying only for what you use.

## Cost Optimization Strategies

Premium SSD v2 uses a **three-dimensional pricing model** where you pay separately for:

1. **Provisioned capacity** (per GiB)
2. **Provisioned IOPS** (above the 3,000 baseline)
3. **Provisioned throughput** (above the 125 MB/s baseline)

This granular billing model creates several optimization opportunities for government IT teams:

### Right-Size from Day One

Because you are not locked into fixed tiers, you can provision exactly the capacity and performance you need. Start with your actual workload requirements and scale up only when monitoring data shows you need more.

### Adjust Performance Without Downtime

Premium SSD v2 allows up to four performance adjustments within any 24-hour period. Note that creating the disk counts as one of these adjustments, so during the first 24 hours after provisioning you can make up to three additional changes. This means you can increase IOPS during end-of-month reporting cycles or batch processing windows, then scale back down to reduce costs during quieter periods. Use the [az disk update](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-deploy-premium-v2#adjust-disk-performance) command or the Azure Portal to make adjustments.

### Eliminate Disk Striping Overhead

With traditional Premium SSDs, agencies often stripe multiple disks together to achieve the IOPS they need. This adds management complexity, increases the risk of configuration errors, and complicates backup strategies. A single Premium SSD v2 disk with up to 80,000 IOPS can replace many striped configurations, simplifying operations and reducing administrative overhead.

### Leverage the Free Baseline

Every Premium SSD v2 disk includes 3,000 IOPS and 125 MB/s at no extra cost. For workloads with moderate performance requirements, this baseline alone may be sufficient, meaning you only pay for capacity.

For current pricing details, visit the [Azure Managed Disks pricing page](https://azure.microsoft.com/en-us/pricing/details/managed-disks/).

## Ideal Government Workloads

Premium SSD v2 is well-suited for a range of public sector scenarios:

- **SQL Server and Oracle databases**: Government ERP systems, financial management platforms, and citizen services databases benefit from the high IOPS and sub-millisecond latency. Oracle Database users should note that release 12.2 or later is required to support the default 4K sector size, though a 512E sector size option is also available.
- **SAP environments**: Agencies running SAP on Azure Government can take advantage of the performance headroom and granular tuning capabilities.
- **Big data and analytics**: Workloads that ingest and process large volumes of data, such as public safety analytics or health data platforms, can benefit from the high throughput ceiling of 1,200 MB/s.
- **MongoDB and Cassandra**: NoSQL databases supporting modern citizen-facing applications perform well on Premium SSD v2 due to the consistent low-latency profile.

## Deployment Considerations

Before deploying Premium SSD v2 in your Azure Government environment, keep these technical considerations in mind:

- **Data disks only**: Premium SSD v2 cannot be used as an OS disk. Continue using Premium SSD or Standard SSD for your OS disks.
- **Zonal VMs required (where applicable)**: In US Gov Virginia, Premium SSD v2 disks can only be attached to VMs deployed in a specific availability zone. Specify the zone when creating the VM.
- **No host caching**: Premium SSD v2 does not support host caching. However, the inherently low latency of the disk compensates for this, and most workloads will see equivalent or better performance.
- **Compatible VM series**: Premium SSD v2 works with a broad range of VM families including DSv3, Ddsv4, Dsv5, ESv3, Edsv5, M-series, and many others. See the [full compatibility list](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types#premium-ssd-v2) for details.
- **Azure Compute Gallery**: Premium SSD v2 disks are not compatible with Azure Compute Gallery images.

### Quick Start with Azure CLI

Here is a concise example of creating a Premium SSD v2 disk in US Gov Arizona and attaching it to an existing VM:

```bash
# Create a Premium SSD v2 disk
az disk create \
  --name myPremiumV2Disk \
  --resource-group myResourceGroup \
  --size-gb 256 \
  --disk-iops-read-write 5000 \
  --disk-mbps-read-write 200 \
  --location usgovarizona \
  --sku PremiumV2_LRS

# Attach to an existing VM
az vm disk attach \
  --vm-name myVM \
  --resource-group myResourceGroup \
  --name myPremiumV2Disk
```

For PowerShell-based deployments and Portal walkthroughs, see the [full deployment guide](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-deploy-premium-v2).

## Why This Matters for Government

Storage performance directly impacts the responsiveness and reliability of the applications that serve citizens and support government operations. Here is why Premium SSD v2 availability in Azure Government is noteworthy for public sector organizations:

**Compliance-ready infrastructure**: Azure Government maintains FedRAMP High authorization, DoD SRG IL4 and IL5 provisional authorizations, and is operated by screened US persons within physically isolated US-based datacenters. Premium SSD v2 inherits all of these compliance assurances. See the [Azure Government overview](https://learn.microsoft.com/en-us/azure/azure-government/documentation-government-overview) for details on the isolation and compliance model.

**Budget efficiency through precision provisioning**: Government procurement cycles are long and budgets are tightly controlled. The ability to provision exactly the storage performance you need, rather than over-provisioning to meet a fixed tier, translates directly into cost savings that matter in the public sector.

**Operational simplicity**: Reducing the number of disks in a striped configuration, eliminating burst-credit management, and having a single disk that can be tuned on demand all reduce the operational burden on government IT teams that are often resource-constrained.

**Mission-critical performance**: Whether it is a public safety database, a financial management system, or a citizen-facing application, the sub-millisecond latency and high IOPS ceiling of Premium SSD v2 ensure that performance is never the bottleneck for critical government services.

**Flexibility for evolving workloads**: Government application landscapes change. A system that handles moderate load today may need significantly more performance during a seasonal surge or a new program rollout. Premium SSD v2 lets you respond to these changes in minutes, not procurement cycles.

## Getting Started

If your agency is running workloads on Azure Government today, evaluating Premium SSD v2 is straightforward:

1. **Audit current disk usage**: Use [Azure Monitor](https://learn.microsoft.com/en-us/azure/azure-monitor/overview) to identify disks where you are over-provisioned on capacity or under-provisioned on IOPS.
2. **Identify candidates**: Workloads with high IOPS requirements but modest capacity needs are the best initial candidates for migration.
3. **Test in a non-production environment**: Create a Premium SSD v2 disk, attach it to a test VM, and benchmark against your current configuration.
4. **Plan your migration**: Since Premium SSD v2 cannot be used as an OS disk, focus your migration on data disks for databases and application storage.

Azure Premium SSD v2 in Azure Government represents a meaningful step forward in giving public sector organizations the storage performance they need with the cost control they require. For agencies looking to modernize their infrastructure while maintaining strict compliance, this is a capability worth evaluating today.

## Additional Resources

- [Azure Managed Disk types overview](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-types)
- [Deploy a Premium SSD v2 disk](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-deploy-premium-v2)
- [Azure Managed Disks pricing](https://azure.microsoft.com/en-us/pricing/details/managed-disks/)
- [Azure Government documentation](https://learn.microsoft.com/en-us/azure/azure-government/documentation-government-overview)
- [Products available by region in Azure Government](https://azure.microsoft.com/en-us/global-infrastructure/services/?products=all&regions=usgov-non-regional,us-dod-central,us-dod-east,usgov-arizona,usgov-texas,usgov-virginia)
