---
title: 'AKS Node OS Migration: What to Do Before the Flatcar Container Linux Deadline'
date: 2026-03-17T13:16:26+00:00
author: Mike Hacker
tags:
- App Modernization
- Announcements
- How To
- Security
categories:
- Azure Kubernetes Service
summary: Flatcar Container Linux for AKS reaches end-of-support on June 8, 2026 - here's how government DevOps teams can identify affected clusters and migrate to Azure Linux or Ubuntu before the deadline.
draft: false
image_prompt: A dramatic cinematic illustration of a fleet of glowing blue containerized server pods migrating across a vast digital landscape from a fading crumbling platform on the left to a strong illuminated Azure-branded logo (no text) platform on the right, with bright energy trails connecting them, deep blue and cyan color palette with volumetric lighting and cloud atmosphere.
image: cover.jpg
---

If your organization has been running Azure Kubernetes Service (AKS) workloads and you happened to experiment with Flatcar Container Linux during its preview period, there's an important deadline you need to act on: **June 8, 2026**. That's when Microsoft ends support for Flatcar Container Linux as an AKS node OS. Three months later, on September 8, 2026, all remaining Flatcar node images are deleted from the platform - meaning any node pool still running Flatcar will lose the ability to scale, reimage, or recover from node failures.

For government DevOps and platform teams who take infrastructure reliability seriously, this is not the kind of surprise you want to encounter mid-incident. This post will walk you through what Flatcar was, how to find out if you're affected, and exactly how to migrate to a fully supported alternative before time runs out.

## What Was Flatcar Container Linux on AKS?

Flatcar Container Linux is an open-source, immutable Linux distribution originally developed by Kinvolk (now part of Microsoft) and maintained by the [Flatcar community](https://www.flatcar.org/). Its design philosophy centers on an immutable OS filesystem that prevents configuration drift and unauthorized changes - appealing characteristics for security-conscious teams.

Microsoft added Flatcar as a **preview** node OS option for AKS, allowing platform teams to deploy node pools using `--os-sku Flatcar`. It offered cross-cloud container portability and an automatic A/B partition update mechanism for OS upgrades. However, since it was only ever available in preview status, it came with a number of significant limitations - limitations that made it particularly unsuitable for production government workloads:

- **No FIPS 140-2 support** - a critical requirement for FedRAMP, CJIS, IRS 1075, and many state-level compliance frameworks
- **No Trusted Launch** - limiting secure boot and vTPM capabilities
- **No Confidential VM support** - restricting access to hardware-based isolation options
- **No SecurityPatch upgrade channel** - meaning you couldn't apply security patches without a full node image upgrade
- **No Azure Monitor VM extension support** - complicating observability in government SOC environments
- **No Node Auto-Provisioning (Karpenter)** - limiting cost optimization and scaling flexibility
- **Generation 1 VMs not supported** - restricting VM size options

Given these limitations, any government team using Flatcar in production was already operating outside best practices. The retirement is an opportunity to move to a supported, compliant baseline.

## The Retirement Timeline

Here are the critical dates you need to plan around:

| Date | Event |
|------|-------|
| **June 8, 2026** | AKS stops producing new Flatcar node images. Security patches end. New Flatcar node pools cannot be created. |
| **September 8, 2026** | All existing Flatcar node images are deleted. Scaling, reimage, and redeploy operations fail on any remaining Flatcar node pools. |

There is **no automatic migration**. If you do nothing, your node pools will stop functioning correctly. The June 8 date is your last chance to take action before the situation becomes critical.

For the official retirement details, see the [Flatcar Container Linux for AKS documentation](https://learn.microsoft.com/en-us/azure/aks/flatcar-container-linux-for-aks).

## Step 1: Identify Affected Clusters and Node Pools

The first action every platform team should take is an audit of all AKS clusters across all subscriptions. If your organization runs both Azure Commercial and Azure Government environments, check both.

Run the following Azure CLI command against each subscription to list all node pools and their OS SKU:

```bash
# List all node pools across a cluster with OS SKU info
az aks nodepool list \
  --resource-group <resource-group-name> \
  --cluster-name <aks-cluster-name> \
  --query '[].{NodePool: name, OsSku: osSku, NodeImageVersion: nodeImageVersion}' \
  -o table
```

You're looking for any node pool where `OsSku` shows `Flatcar` or where `NodeImageVersion` contains `AKSFlatcar`. For example, an affected node pool might show a node image version like `AKSFlatcar-flatcargen2-202508.06.0`.

To sweep across multiple subscriptions at once, you can use a script like this:

```bash
# Audit all subscriptions for Flatcar node pools
for sub in $(az account list --query '[].id' -o tsv); do
  echo "Checking subscription: $sub"
  az account set --subscription $sub
  for rg in $(az group list --query '[].name' -o tsv); do
    for cluster in $(az aks list --resource-group $rg --query '[].name' -o tsv 2>/dev/null); do
      az aks nodepool list \
        --resource-group $rg \
        --cluster-name $cluster \
        --query "[?osSku=='Flatcar'].{Cluster:'$cluster', NodePool: name, OsSku: osSku}" \
        -o table 2>/dev/null
    done
  done
done
```

Document every affected cluster, including the resource group, subscription ID, and whether it's in Azure Commercial or Azure Government.

## Step 2: Choose Your Migration Target OS

Microsoft supports two Linux OS options for AKS node pools: **Azure Linux** and **Ubuntu**. For government workloads, Azure Linux is the recommended choice, and here's why.

### Azure Linux - The Recommended Choice for Government

[Azure Linux](https://learn.microsoft.com/en-us/azure/azure-linux/intro-azure-linux) (based on CBL-Mariner) is a Microsoft-maintained, purpose-built container host OS that was designed from the ground up for AKS. Azure Linux 3.0 is generally available with the following major components:

- **Kernel 6.6**
- **containerd 2.0**
- **systemd 255**
- **SymCrypt** cryptographic library

For government teams specifically, Azure Linux delivers several compelling advantages:

**Smallest attack surface on AKS** - Azure Linux includes only ~500 packages, taking up to 5 GB less disk space than other OS options. Fewer packages means fewer CVEs to track, fewer patches to apply, and a smaller compliance scope.

**CIS Level 1 Benchmark compliance** - Azure Linux is the **only** Linux distribution on AKS that passes all [CIS Level 1 benchmarks](https://learn.microsoft.com/en-us/azure/aks/cis-azure-linux). For government teams navigating NIST SP 800-53, FedRAMP, or state-specific hardening requirements, this is a significant baseline advantage.

**FIPS 140-2 support** - Azure Linux node pools support FIPS-enabled images, which is a requirement for many federal and state compliance frameworks. This can be enabled with the `--enable-fips-image` flag when creating node pools.

**Secure supply chain** - Every Azure Linux package is built from source, signed, and validated by Microsoft teams before release. Packages and sources are hosted on Microsoft-owned, secured platforms - an important consideration for supply chain security requirements under frameworks like NIST SP 800-161.

**Microsoft-owned lifecycle** - Unlike community-maintained distributions, Azure Linux patches are released monthly for standard CVEs, with critical updates issued within days. There's no dependency on a third-party community to maintain the OS you're running in production.

### Ubuntu - The Familiar Alternative

Ubuntu remains fully supported on AKS and is the default Linux OS. If your workloads have Ubuntu-specific dependencies or your team has deep Ubuntu expertise, it's a completely valid migration target. Ubuntu node pools also support FIPS-enabled images. However, note that **Ubuntu 20.04 is being retired from AKS on March 17, 2027** - so if you migrate to Ubuntu, ensure you're targeting Ubuntu 22.04 or later.

## Step 3: Migrate Your Node Pools

This is the most important technical detail to understand: **there is no in-place migration path from Flatcar to another OS**. Unlike Ubuntu-to-Azure Linux migrations (which support in-place OS SKU migration), Flatcar requires a full node pool replacement. Your migration process will follow this pattern:

### Migration Process Overview

**1. Create a new node pool with your target OS**

Add a new system node pool running Azure Linux to your existing cluster:

```bash
az aks nodepool add \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name azlinuxpool \
  --mode System \
  --os-sku AzureLinux \
  --node-count 3
```

For Ubuntu, replace `--os-sku AzureLinux` with `--os-sku Ubuntu`.

If you need FIPS compliance on the new node pool:

```bash
az aks nodepool add \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name fipspool \
  --mode User \
  --os-sku AzureLinux \
  --enable-fips-image \
  --node-count 3
```

**2. Validate workload compatibility**

Before moving production workloads, deploy a representative set of your applications to the new node pool and validate behavior. Use node selectors or taints to direct test workloads specifically to the new pool. Pay particular attention to any workloads that make assumptions about the underlying OS, interact with host paths, or use OS-level security features.

**3. Migrate workloads**

Cordon the Flatcar node pool nodes to prevent new pods from scheduling there:

```bash
# Cordon all nodes in the Flatcar pool
for node in $(kubectl get nodes -l agentpool=<flatcar-pool-name> -o name); do
  kubectl cordon $node
done
```

Then drain the nodes to evict existing pods (ensure you have appropriate [Pod Disruption Budgets](https://kubernetes.io/docs/tasks/run-application/configure-pdb/) configured to manage availability during the drain):

```bash
for node in $(kubectl get nodes -l agentpool=<flatcar-pool-name> -o name); do
  kubectl drain $node --ignore-daemonsets --delete-emptydir-data
done
```

**4. Delete the Flatcar node pool**

Once all workloads have migrated and you've verified they're running healthy on the new pool:

```bash
az aks nodepool delete \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <flatcar-pool-name>
```

**5. Configure automatic node image upgrades**

After migrating, set up automatic node image upgrades to keep your new node pools current:

```bash
az aks update \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --auto-upgrade-channel node-image
```

## Azure Government Considerations

AKS is available in Azure Government regions (US Gov Virginia, US Gov Arizona, US Gov Texas), and both Azure Linux and Ubuntu node pools are supported in Azure Government. If you're running AKS workloads in `AzureUSGovernment` cloud environments, the same migration steps apply - simply ensure your Azure CLI is configured to target the correct cloud:

```bash
az cloud set --name AzureUSGovernment
az login
```

One area to verify in Azure Government deployments is feature parity. Some newer AKS features may have a delayed rollout to government regions compared to commercial Azure. Before planning your migration timeline, confirm that the specific Azure Linux features you intend to use (such as FIPS-enabled images, Trusted Launch, or specific VM sizes) are available in your target government region using the [Azure Government feature availability documentation](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure).

For Azure Government workloads with FedRAMP High requirements, Azure Linux's FIPS 140-2 support and CIS Level 1 benchmark compliance provide a strong foundation for your compliance documentation. The secure supply chain model - where Microsoft builds, signs, and validates every package from source - aligns well with supply chain security controls in NIST SP 800-53 Rev 5.

## Why This Matters for Government

Government organizations operate under a level of scrutiny and compliance obligation that makes OS-level decisions consequential in ways that don't apply to commercial organizations. A few key reasons why this migration deserves dedicated attention from government IT leadership:

**Compliance risk from end-of-support software** - Running an OS with no active security patches violates the fundamental principles of FedRAMP's Vulnerability Scanning and Patch Management controls. After June 8, Flatcar receives no security updates. After September 8, it becomes operationally unviable. Any government workload still running on Flatcar past these dates will be out of compliance with virtually every federal and state security framework.

**Preview software in production** - Flatcar was only ever a preview feature on AKS. Government technology acquisition and deployment policies typically prohibit running preview or beta software in production environments. This retirement is an opportunity to audit whether preview AKS features are in use and establish controls to prevent preview-to-production slippage.

**Azure Linux is the right long-term foundation** - The combination of minimal package footprint, Microsoft-managed lifecycle, FIPS support, CIS Level 1 benchmark compliance, and secure supply chain makes Azure Linux the most compliance-friendly AKS node OS available. Government teams that migrate to Azure Linux gain a node OS that was purpose-built with security and reliability as primary design goals - not afterthoughts.

**Operational continuity** - The September 8, 2026 hard deadline for image deletion means that any Flatcar node pool that experiences a node failure after that date cannot be automatically remediated. In a mission-critical government application, this creates unacceptable operational risk. Migration needs to happen well before the June 8 soft deadline to allow for testing and validation.

## Action Items and Timeline

Here's a recommended action plan for government platform teams:

1. **Immediately**: Audit all AKS clusters across Commercial and Government subscriptions; identify all Flatcar node pools
2. **April 2026**: Stand up test clusters running Azure Linux; validate key workloads; establish migration runbooks
3. **May 2026**: Begin migrating non-production environments to Azure Linux node pools
4. **By June 7, 2026**: Complete all production migrations; delete remaining Flatcar node pools

Don't wait until the June 8 deadline. Build in buffer time for validation, change control approval processes (which government organizations typically require), and any unforeseen workload compatibility issues.

## Additional Resources

- [Flatcar Container Linux for AKS - Retirement Notice](https://learn.microsoft.com/en-us/azure/aks/flatcar-container-linux-for-aks)
- [Azure Linux on AKS Overview](https://learn.microsoft.com/en-us/azure/azure-linux/intro-azure-linux)
- [Migrate Node Pools to Azure Linux](https://learn.microsoft.com/en-us/azure/azure-linux/tutorial-azure-linux-migration)
- [Enable FIPS on AKS Node Pools](https://learn.microsoft.com/en-us/azure/aks/enable-fips-nodes)
- [AKS Node Images Documentation](https://learn.microsoft.com/en-us/azure/aks/node-images)
- [AKS Release Notes and Announcements](https://github.com/Azure/AKS/releases)
- [Azure Government - AKS Feature Availability](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure)
- [CIS Azure Linux Benchmarks](https://learn.microsoft.com/en-us/azure/aks/cis-azure-linux)
