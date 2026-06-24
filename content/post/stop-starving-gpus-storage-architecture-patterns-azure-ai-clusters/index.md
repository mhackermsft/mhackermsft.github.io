---
title: 'Stop Starving Your GPUs: Storage Architecture Patterns That Keep AI Clusters Fully Utilized on Azure'
date: 2026-06-24T11:17:40+00:00
author: Mike Hacker
tags:
- AI
- Data Platform
- How To
categories:
- Azure
- AI
summary: A technical deep-dive on reducing data-pipeline starvation for GPU training and inference using Azure Managed Lustre, Azure NetApp Files, and premium Blob Storage.
draft: false
image_prompt: Technical illustration of an Azure GPU cluster fed by Azure Managed Lustre, Azure NetApp Files, and premium Blob Storage, emphasizing high-throughput data flow, checkpointing, and public-sector cloud governance.
image: cover.png
audio: audio.mp3
---

GPU capacity is one of the highest-cost resources in many government AI programs. Reserved or committed GPU capacity costs money whether the accelerators are processing batches or waiting for data. When training and inference deployments waste that capacity, the root cause is often not the model architecture. It is the storage path that cannot feed data fast enough.

This post walks through storage architecture patterns that help keep Azure GPU clusters busy. The focus is on three Azure services that address different parts of the problem: Azure Managed Lustre for high-throughput training, Azure NetApp Files for persistent low-latency file workloads, and premium block blob storage as a durable object storage tier.

## Why GPUs Starve

A GPU training step has a simple rhythm: read a batch of data, run forward and backward passes, update weights, repeat. If the storage and data-loading path cannot deliver the next batch before the current step finishes, the GPU waits on I/O. In practice, you can often see this as utilization that alternates between high activity and idle periods.

The three pressure points are usually:

1. **Aggregate read throughput** during training, when many workers stream images, text shards, feature files, or embeddings at the same time.
2. **Checkpoint write bursts**, when model state is flushed to durable storage on a schedule.
3. **Metadata and small-file overhead**, where high file counts and random reads create pressure that simple object access patterns may not absorb efficiently.

The right answer is usually a combination of services, not a single storage product for every phase of the AI lifecycle.

## Option 1: Azure Managed Lustre for High-Throughput Training

[Azure Managed Lustre](https://learn.microsoft.com/en-us/azure/azure-managed-lustre/amlfs-overview) is a managed parallel file system for high-performance computing workloads. It provides Lustre protocol compatibility and integrates with Azure Blob Storage through Lustre hierarchical storage management, or HSM.

The key sizing decision is the SKU. Azure Managed Lustre durable SSD SKUs scale throughput with provisioned capacity. The documented SKU options are:

| SKU | Throughput per TiB | Storage range |
|-----|-------------------|---------------|
| AMLFS-Durable-Premium-40 | 40 MB/s | 48 - 768 TB |
| AMLFS-Durable-Premium-125 | 125 MB/s | 16 - 128 TB |
| AMLFS-Durable-Premium-250 | 250 MB/s | 8 - 128 TB |
| AMLFS-Durable-Premium-500 | 500 MB/s | 4 - 128 TB |

That relationship is the core sizing model. A 100 TiB file system on the 500 MB/s-per-TiB SKU provides roughly 50 GB/s of aggregate throughput. If the dataset is smaller than the throughput target requires, provisioning additional capacity to buy throughput can be a valid architecture decision, provided the cost is modeled against GPU idle time.

Blob integration is the second design decision. Instead of keeping every dataset on the parallel file system between runs, store the canonical copy in Azure Blob Storage, hydrate the namespace into Azure Managed Lustre, run the training job, export changed files or checkpoints back to Blob Storage, and then delete the file system when the campaign is complete. The pricing model is pay-as-you-go for the provisioned Azure Managed Lustre file system, so the cost-control pattern is to keep the file system provisioned only for the period when the workload needs it.

Here is a Bicep example using the current Azure Managed Lustre resource reference and Blob integration settings:

```bicep
resource filesystem 'Microsoft.StorageCache/amlFilesystems@2026-01-01' = {
  name: 'amlfs-training'
  location: 'eastus'
  sku: {
    name: 'AMLFS-Durable-Premium-250'
  }
  properties: {
    filesystemSubnet: '<subnet-resource-id>'
    storageCapacityTiB: 16
    maintenanceWindow: {
      dayOfWeek: 'Saturday'
      timeOfDayUTC: '22:00'
    }
    hsm: {
      settings: {
        container: '<blob-container-resource-id>'
        loggingContainer: '<logging-container-resource-id>'
        importPrefixesInitial: [ '/' ]
      }
    }
  }
  zones: [ '1' ]
}
```

For containerized training, Azure Managed Lustre supports an AKS-compatible CSI driver. That lets Kubernetes workloads mount the Lustre namespace through the supported driver rather than embedding storage setup logic inside each training container.

## Option 2: Azure NetApp Files for Mixed and Latency-Sensitive Workloads

Not every AI workload is a massive distributed training run. Inference services, fine-tuning jobs, feature stores, shared research workspaces, and model registries often need persistent file access, low latency, snapshots, and replication more than maximum aggregate training throughput.

[Azure NetApp Files](https://learn.microsoft.com/en-us/azure/azure-netapp-files/azure-netapp-files-introduction) is an Azure native, first-party file storage service backed by NetApp technology. It supports NFS, SMB, dual-protocol access, and data-management features such as snapshots and replication.

Azure NetApp Files supports Standard, Premium, Ultra, Flexible, and Elastic service levels. Elastic zone-redundant storage is documented as public preview, so production designs should confirm preview acceptability before using it for regulated workloads. Dynamic service-level changes are supported by moving a volume to another capacity pool in the same NetApp account, but there are constraints. For example, Flexible capacity pools cannot be converted to Standard, Premium, or Ultra, and Standard, Premium, and Ultra pools cannot be converted to Flexible.

For inference fleets that read model weights and write embeddings, Premium or Ultra can provide predictable low-latency file access. Cool access is available with the Flexible, Standard, Premium, and Ultra service levels and can move inactive data blocks to a cool tier while keeping the namespace available to applications.

Large volumes support sizes from 50 TiB to 1,024 TiB by default. Larger modes, including breakthrough mode and large volumes up to 7.2 PiB with cool access, are documented as preview and require request or waitlist steps, so treat them as design options that need explicit regional and program approval rather than default assumptions.

The practical pattern is to use Azure NetApp Files for the persistent, long-lived, latency-sensitive data plane: model registries, fine-tuning datasets, feature stores, and checkpoints that benefit from snapshots or replication. Azure NetApp Files snapshots can support fast recovery, and application consistency can be achieved when snapshots are coordinated with the application layer or appropriate tooling.

## Option 3: Premium Block Blob as the Durable Backbone

Azure Blob Storage is the durable object storage tier for unstructured data. For AI workloads with high transaction rates or latency-sensitive object access, [premium block blob storage accounts](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blob-block-blob-premium) store data on SSDs and are optimized for low latency and high transaction rates. Microsoft documentation calls out AI and machine learning as example workloads.

Premium block blob storage has a higher storage cost than standard general-purpose v2 accounts, but lower transaction costs. It can be cost-effective for workloads with high transaction density, but the break-even point depends on transaction mix, region, redundancy, and data volume. Use the Azure pricing calculator or the Blob Storage pricing page before treating premium as the default.

When Linux GPU nodes need to read directly from Blob Storage through a file-system interface, use BlobFuse2. The mount command supports `--config-file`, `--tmp-path`, and `--read-only`. The `--tmp-path` option configures the file-cache location, and the documentation recommends the fastest disk, such as SSD or ramdisk, for best performance. On Azure Linux VMs, the temporary disk is typically exposed under `/mnt/resource`; confirm the VM size and local disk behavior before relying on that path for cache performance.

```bash
sudo mkdir -p /mnt/resource/blobfuse2tmp
sudo chown $USER /mnt/resource/blobfuse2tmp
mkdir -p ./data
sudo blobfuse2 mount ./data \
  --config-file=./config.yaml \
  --tmp-path=/mnt/resource/blobfuse2tmp \
  --read-only
```

The `--read-only` flag is appropriate for training input datasets where the Blob container is the canonical source and training workers should not write back to it.

## Checkpointing Without Stalling

Checkpoint writes are a common starvation source. A naive training job pauses every rank, writes the full model state synchronously to durable storage, and resumes only after the write completes. That turns every checkpoint interval into GPU idle time.

Two patterns reduce the impact:

1. **Write checkpoints to the fast file system first, then archive.** Flush checkpoints to Azure Managed Lustre or another high-throughput file tier, then export to Blob Storage after the write burst is absorbed.
2. **Stagger and shard checkpoint writes.** Avoid forcing every rank to contend for the same metadata path at the same moment. Size the checkpoint interval against measured throughput and recovery-point objectives, not habit.

The same principle applies to data loading. Use enough workers and prefetch depth so the next batch is ready before the current step finishes, then tune against observed GPU utilization and storage metrics.

## Why This Matters for Government

State and local agencies are moving AI from pilots into production use cases such as document processing, constituent services, translation, health analytics, and public safety workflows. These programs often run under budget scrutiny and compliance obligations, so storage design affects both cost and risk.

The storage tier should support three public-sector goals. First, it should lower effective cost per training run by reducing GPU wait time. Second, it should provide predictable performance for citizen-facing inference and operational analytics. Third, it should support governance choices that align with the agency workload, such as encryption, key management, retention, and recovery.

Microsoft documents Azure and Azure Government compliance coverage for frameworks and obligations that commonly matter to public-sector workloads, including [FedRAMP](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-fedramp), [CJIS](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-cjis), and [HIPAA](https://learn.microsoft.com/en-us/azure/compliance/offerings/offering-hipaa-us). Those compliance documents do not make an application compliant by themselves. They help agencies understand Microsoft-responsible controls, shared responsibilities, and the design work the customer must still perform.

Storage-specific controls also matter. Azure Storage encrypts data at rest by default and supports customer-managed keys for Blob Storage. Azure NetApp Files supports customer-managed keys for volume encryption, snapshots, and replication patterns. Azure Managed Lustre encrypts stored data and can integrate with Blob Storage for longer-term redundancy choices. Match those service capabilities to the classification, retention, and recovery requirements of the workload.

One planning note: feature and service availability can differ between Azure commercial and Azure Government regions. Before committing an architecture, confirm that the specific Azure Managed Lustre SKUs, Azure NetApp Files service levels, premium block blob features, and preview features are available in the target region using the official [Azure products-by-region](https://azure.microsoft.com/en-us/explore/global-infrastructure/products-by-region/) page.

## Bringing It Together

A practical reference pattern for a government AI program looks like this: Blob Storage holds canonical datasets and archived checkpoints; Azure Managed Lustre is provisioned for high-throughput training phases and integrated with Blob Storage for import and export; Azure NetApp Files serves persistent, latency-sensitive file workloads such as model registries, feature stores, and shared research data. BlobFuse2 can support direct Blob access where a file-system interface is useful, with local cache placement tuned to the VM.

The accelerators are the scarce resource. Architect the storage path so they spend their time computing instead of waiting for data.
