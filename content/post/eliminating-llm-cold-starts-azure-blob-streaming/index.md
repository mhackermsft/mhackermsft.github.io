---
title: 'Eliminating LLM Cold Starts on Azure: Streaming Model Weights from Blob Storage to GPU Memory'
date: 2026-05-19T15:44:16+00:00
author: Mike Hacker
tags:
- AI
- App Modernization
- How To
categories:
- AI
summary: A deep technical walkthrough on using the Run:ai Model Streamer with Azure Blob Storage to slash LLM cold-start times on Azure GPU VMs, with code, architecture patterns, and cost notes for government workloads.
draft: false
image_prompt: A clean isometric illustration of an Azure datacenter rack with GPU nodes on the right and a stylized Azure Blob Storage container on the left, with thick parallel data streams (multiple colored ribbons) flowing in parallel from the blob into the GPUs. Cool blue and teal Azure palette, government/professional aesthetic, no text.
image: cover.png
audio: audio.mp3
---

Anyone who has stood up a self-hosted large language model on Azure has felt the cold-start tax. You provision a GPU VM, the container pulls a 30-70 GB weights file, the loader memory-maps it, copies it host-side, and only then begins shoveling tensors across PCIe into the GPU. For a Llama-3 70B or Mixtral 8x22B checkpoint, that sequence routinely takes 5-15 minutes per pod. Multiply that by every autoscale event, every spot-VM eviction, every node recycle, and you have a real problem: idle H100s burning budget while operators wait on `torch.load`.

This post walks through a pattern that government AI teams can deploy today on both Azure commercial and Azure US Government: pairing **Azure Blob Storage** as the model registry with the open-source **Run:ai Model Streamer** (Apache-2.0) to stream safetensors weights concurrently from object storage directly into GPU memory. Published benchmarks from the project show roughly 6x faster cold loads versus the default Hugging Face safetensors loader when reading from a high-throughput object store.

## Why default loaders are slow

The stock `safetensors` and `transformers` loaders are essentially single-threaded with respect to a given file. They:

1. Open the file (or download it sequentially from the Hub).
2. Memory-map it into host RAM.
3. Iterate tensor-by-tensor, copying each to GPU with `.to('cuda')`.

On Azure, the bottleneck rarely turns out to be the GPU or even PCIe; it is the **serial read** from whatever filesystem holds the weights. A single HTTPS GET against an Azure Blob is capped well below what a Standard_ND96isr_H100_v5 VM's frontend NIC can sustain, and a single mmap from a Premium SSD v2 disk leaves the H100s idle for minutes.

## What the Run:ai Model Streamer does differently

The [Run:ai Model Streamer](https://github.com/run-ai/runai-model-streamer) is a small Python SDK with a C++ core. It does three things that matter:

- **Concurrent multi-range reads.** It opens many parallel byte-range requests against the object store, saturating NIC throughput rather than relying on one TCP stream. The number of reader threads is controlled by the `RUNAI_STREAMER_CONCURRENCY` environment variable.
- **Overlapping host-to-device copies.** While later chunks are still in flight from storage, earlier chunks are already being DMA'd into GPU memory via pinned host buffers.
- **Native object-store clients.** Native Azure Blob support landed in release 0.15.6 (February 2026) and was extended in 0.15.9 (late April 2026) with `AZURE_STORAGE_ACCOUNT_KEY` credential support via `StorageSharedKeyCredential`, so there is no Python GIL contention on the read path, no FUSE layer required, and no intermediate file on local disk.

The latest release as of this writing is **0.15.9 (late April 2026)**, and the project is integrated as a first-class weight loader inside **vLLM** via the `--load-format runai_streamer` argument, which is the inference server most government teams are standardizing on for open-weight models.

## Reference architecture on Azure

For a government AI inference platform, the pattern looks like this:

- **Model registry:** A dedicated Azure Blob Storage account (general-purpose v2 or premium block blob) in the same region as the GPU compute. Containers are organized as `models/<family>/<version>/` with the safetensors shards as blobs. See the [Azure Blob Storage introduction](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction) for the container/blob model.
- **Compute:** An AKS node pool or VM Scale Set on [ND H100 v5](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/gpu-accelerated/nd-family) (commercial) or the equivalent ND-series SKU available in your Azure US Government region. ND H100 v5 gives you 8 H100 GPUs per VM plus 8x 400 Gb/s NVIDIA Quantum-2 InfiniBand links, which is exactly the kind of fabric you want feeding the streamer.
- **Network path:** A Private Endpoint on the storage account so traffic never traverses the public internet. In Azure US Government, the blob endpoint is `blob.core.usgovcloudapi.net` rather than `blob.core.windows.net`; both support Private Link. See the [Azure Government services compatibility doc](https://learn.microsoft.com/en-us/azure/azure-government/compare-azure-government-global-azure) for the full endpoint mapping.
- **Identity:** A user-assigned managed identity bound to the AKS node pool (workload identity) with the **Storage Blob Data Reader** role on the model container. No SAS tokens to rotate, no keys to leak.
- **Inference runtime:** vLLM (or a custom Triton/PyTorch server) configured to use the Run:ai Model Streamer as the weight loader.

## Getting weights into Blob

Upload your shards with AzCopy or the Azure CLI. For a typical 70B model checkpoint:

```bash
az storage container create \
  --account-name govaiweights \
  --name models \
  --auth-mode login

azcopy login --tenant-id <tenant>
azcopy copy "./llama-3.1-70b-instruct/*" \
  "https://govaiweights.blob.core.usgovcloudapi.net/models/llama-3.1-70b-instruct/v1/" \
  --recursive --block-size-mb 64
```

Use a 64 MB block size. Smaller blocks fragment the object and hurt parallel read efficiency; larger blocks waste memory on the loader side.

## Wiring up the streamer

The minimal Python pattern, straight from the project README:

```python
import os
from runai_model_streamer import SafetensorsStreamer

os.environ["AZURE_STORAGE_ACCOUNT_NAME"] = "govaiweights"
file_path = "az://models/llama-3.1-70b-instruct/v1/model-00001-of-00030.safetensors"

with SafetensorsStreamer() as streamer:
    streamer.stream_file(file_path)
    for name, tensor in streamer.get_tensors():
        gpu_tensor = tensor.to('cuda:0')
```

With the native Azure Blob support added in 0.15.6, the streamer addresses blobs via an `az://<container>/<path>` URI, resolves the account from `AZURE_STORAGE_ACCOUNT_NAME`, and authenticates through `DefaultAzureCredential` (managed identity, `az login`, service principal env vars, etc.). The same scheme works against Azure US Government, since the SDK respects the standard cloud endpoint configuration. If you prefer a filesystem abstraction, you can mount via [BlobFuse2](https://learn.microsoft.com/en-us/azure/storage/blobs/blobfuse2-what-is) and point the streamer at the mount path instead. The native-client path keeps the data plane on the Blob REST API, traverses your Private Endpoint, and authenticates via the workload's managed identity.

For vLLM, the equivalent native invocation skips the mount entirely:

```bash
AZURE_STORAGE_ACCOUNT_NAME=govaiweights \
vllm serve az://models/llama-3.1-70b-instruct/v1 \
  --load-format runai_streamer \
  --tensor-parallel-size 8 \
  --max-model-len 8192
```

If you do prefer BlobFuse2, mount once at pod start with a config file:

```bash
blobfuse2 mount /mnt/models \
  --config-file /etc/blobfuse2.yaml \
  --read-only=true
```

Then point vLLM at the mount with the Run:ai loader enabled:

```bash
vllm serve /mnt/models/llama-3.1-70b-instruct/v1 \
  --load-format runai_streamer \
  --tensor-parallel-size 8 \
  --max-model-len 8192
```

The `--load-format runai_streamer` flag is the integration point. vLLM hands the directory to the streamer, the streamer opens parallel range reads against every shard, and the H100s start receiving tensors within seconds rather than minutes.

## Tuning that actually moves the needle

A few knobs matter more than the rest:

- **Concurrency.** Tune `RUNAI_STREAMER_CONCURRENCY` (or vLLM's `--model-loader-extra-config '{"concurrency":N}'`) and consider `RUNAI_STREAMER_MEMORY_LIMIT` to cap the pinned CPU buffer. On an 8-GPU ND H100 v5, a few dozen reader threads per shard is a reasonable starting point. Too few leaves bandwidth on the floor; too many starts hitting blob throttle limits and wasting host memory on pinned buffers. Note that distributed streaming (one reader per rank, with broadcast) is enabled by default when streaming from object storage to CUDA devices and can be toggled with `RUNAI_STREAMER_DIST`.
- **Storage account scale targets.** Per the [Azure Storage scalability targets](https://learn.microsoft.com/en-us/azure/storage/common/scalability-targets-standard-account), a single standard GPv2 storage account in a tier-1 region defaults to 40,000 requests per second, 60 Gbps ingress, and 200 Gbps egress. In smaller regions the defaults are 20,000 RPS and 50 Gbps egress. If you are warming many pods simultaneously and bumping into those ceilings, shard your model across multiple storage accounts, request a quota increase, or move to a Premium Block Blob account for sustained low-latency reads.
- **Co-locate compute and storage.** Always deploy the storage account in the same Azure region as the GPU node pool. Cross-region reads will dominate your cold-start time and rack up egress charges.
- **Pre-pull is not the answer.** Resist the temptation to bake weights into the container image. A 140 GB image cripples your registry, your nodes, and your patch cadence. Keep the image lean and let the streamer do its job at pod start.

## Cost implications

The economics for a government AI platform are straightforward. An ND H100 v5 VM is one of the most expensive SKUs on Azure. Every minute that VM sits in cold start is pure waste, and at typical list prices a 10-minute cold start across a 4-node pool costs more than a month of Blob Storage for the weights themselves. Cutting cold start from roughly 10 minutes to roughly 90 seconds:

- Reclaims roughly 85% of the otherwise-wasted GPU minutes on every scale event.
- Makes spot/low-priority GPU capacity actually viable, because eviction recovery is fast enough to absorb.
- Lets you scale-to-zero overnight for non-24x7 agencies (think permitting portals, constituent chatbots) without punishing the first morning user with a multi-minute wait.

Blob Storage itself is cheap by comparison: even at Hot tier pricing, hosting a few hundred gigabytes of weights is rounding error against a single GPU-hour.

## Why This Matters for Government

State and local agencies adopting open-weight LLMs face three pressures simultaneously: tight budgets, FedRAMP/CJIS data-handling constraints, and unpredictable demand curves. This pattern addresses all three.

- **Budget.** Faster cold starts mean you can right-size GPU pools to actual demand and use autoscaling aggressively. You stop paying for idle H100s waiting on a loader.
- **Data residency and sovereignty.** Weights stay in your Azure Blob account, inside your VNet, behind your Private Endpoint, in either Azure commercial or Azure US Government. Nothing transits to the Hugging Face Hub at request time, nothing is cached on a third-party CDN, and managed identity removes the need to store credentials on the node. For agencies in the M365 GCC tenant who are running inference in Azure US Government, the same code works against `blob.core.usgovcloudapi.net` with no application changes.
- **Operational resilience.** When a node fails or a spot VM is reclaimed at 2 a.m., the replacement pod is serving traffic in under two minutes instead of fifteen. That is the difference between a non-event and a paged on-call engineer.
- **Model governance.** Treating Blob as the authoritative model registry gives you a single place to apply immutable blob policies, lifecycle management, RBAC, and Defender for Storage threat protection. Model versions become first-class artifacts that auditors can reason about.

## Where to go next

The Run:ai Model Streamer is open source under Apache-2.0 and lives at [github.com/run-ai/runai-model-streamer](https://github.com/run-ai/runai-model-streamer); the 0.15.9 release from late April 2026 is the current stable. The vLLM integration is documented in [Loading models with Run:ai Model Streamer](https://docs.vllm.ai/en/stable/models/extensions/runai_model_streamer.html). Pair those with the [Azure Blob Storage documentation](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction), an [ND H100 v5 node pool](https://learn.microsoft.com/en-us/azure/virtual-machines/sizes/gpu-accelerated/nd-family), and workload identity, and you have a production-grade, sovereign-cloud-friendly inference platform whose cold starts are measured in seconds rather than coffee breaks.
