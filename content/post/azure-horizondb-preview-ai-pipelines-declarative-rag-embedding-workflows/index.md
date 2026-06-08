---
title: 'AI Pipelines in Azure HorizonDB Preview: Declarative RAG and Embedding Workflows That Live in the Database'
date: 2026-06-08T14:20:38+00:00
author: Mike Hacker
tags:
- Data Platform
- AI
- Announcements
categories:
- Azure
summary: A technical look at declarative AI pipelines in Azure HorizonDB preview, which run chunking, embedding, and durable RAG ingestion workflows inside PostgreSQL, and what they mean for government data platforms.
draft: false
image_prompt: A cinematic photograph inside a sleek, brightly lit modern data center, a single tall server rack glowing at the center with cool blue and white light. Luminous ribbons of light flow inward carrying small translucent document-like fragments that dissolve into glowing blue points of light, which arrange themselves into a neat three-dimensional lattice suspended within the rack's transparent core. Shallow depth of field, dramatic rim lighting, polished floor reflections. No text, letters, numbers, or writing anywhere in the image.
image: cover.png
audio: audio.mp3
---

Most retrieval-augmented generation (RAG) projects in government start the same way: someone stands up a separate service in the application tier that reads rows from a database, calls an embedding model, and writes vectors back. It works in a demo. Then it meets production, where a transient API failure mid-batch leaves no shared checkpoint, a worker crashes after writing some chunks but before marking the parent row processed, and a model upgrade leaves no clean way to re-embed exactly the rows that changed.

Azure HorizonDB, the cloud-native managed PostgreSQL service Microsoft has in public preview, takes a different position: the embedding pipeline belongs close to the data, defined declaratively in SQL, and executed durably. The **AI pipelines** capability, documented in the June 2026 Microsoft Learn refresh, lets a team describe a chunk, embed, index, and rank workflow as a database object that survives crashes, retries failed steps, and resumes long-running jobs from the last completed step.

This post walks through how it works, why the architecture matters, and the availability caveats that government IT leaders need to weigh before planning around it.

## What Azure HorizonDB is, briefly

Before the pipelines, the platform. Per the [What is Azure HorizonDB?](https://learn.microsoft.com/en-us/azure/horizondb/overview) overview, HorizonDB is a fully managed, PostgreSQL-compatible database built on two architectural principles: disaggregated compute and storage, and a database-as-a-log design where only the write-ahead log (WAL) is written from compute to a purpose-built, zone-resilient storage fleet. Compute replicas are stateless, read replicas provision quickly because they share the same durable storage, and failover is faster because no log rewinding is needed.

For an AI workload, the relevant point is that your operational data, vector embeddings, JSON, and graph relationships can live in one transactional system that already has point-in-time recovery, high availability, and security controls. You do not have to synchronize a separate vector store against a system of record for the ingestion path. The database can be the system of record and the retrieval substrate.

## The pipeline anatomy

AI pipelines are part of the `azure_ai` extension and are built on top of `pg_durable`, a general durable-execution engine. Where `pg_durable` gives you raw durable functions, the `ai.*` pipeline API gives you a higher-level, AI-shaped surface that compiles down to a durable graph automatically. A pipeline has four parts, per the [Implement AI pipelines](https://learn.microsoft.com/en-us/azure/horizondb/ai/ai-pipelines) guide:

- **Source**: where rows come from. Today that is a `table_source(...)` over a HorizonDB table, optionally with an `incremental_column` so the pipeline skips rows it has already processed.
- **Steps**: the AI operations applied to each row, in order. The available step types are `ai.chunk()`, `ai.embed()`, `ai.extract()`, `ai.generate()`, and `ai.rank()`.
- **Sink**: an optional destination for results.
- **Trigger**: `'on_change'` to run automatically when source rows change, or `'manual'` to run only when you call `ai.run()`.

The definition is just a row in a system catalog. Creating it registers the workflow but runs nothing.

## A minimal RAG ingestion pipeline

First, enable the required extensions and define both the source and sink tables. On HorizonDB, extensions must be allowed at the instance level before they are created in a database. The `pg_diskann` extension depends on `vector`, so create `vector` first or use the documented `CASCADE` option.

```sql
CREATE EXTENSION IF NOT EXISTS pg_durable;
CREATE EXTENSION IF NOT EXISTS azure_ai;
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS pg_diskann;

CREATE TABLE documents_ai_pipeline (
    id         SERIAL PRIMARY KEY,
    title      TEXT NOT NULL,
    content    TEXT NOT NULL,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE documents_ai_pipeline_output (
    doc_id      INT,
    chunk_index INT,
    chunk_text  TEXT,
    embedding   vector(1536),
    metadata    JSONB
);

CREATE INDEX diskann_sq_embedding_idx
    ON documents_ai_pipeline_output
    USING diskann (embedding vector_cosine_ops)
    WITH (spherical_quantized = true);
```

The sink table contract is important: the AI pipelines documentation specifies `doc_id`, `chunk_index`, `chunk_text`, `embedding`, and `metadata` for the output table.

Then declare the pipeline. This one chunks a `content` column, generates embeddings, and writes to the sink, re-running automatically whenever source rows change:

```sql
SELECT ai.create_pipeline(
    name   => 'rag_pipeline',
    source => ai.table_source(table_name => 'documents_ai_pipeline'),
    steps  => ARRAY[
        ai.chunk(input => 'content', chunk_size => 512, overlap => 64),
        ai.embed(model => 'default-embedding',
                 input => 'chunk_text',
                 dimensions => 1536)
    ],
    trigger => 'on_change',
    sink    => ai.table_sink('documents_ai_pipeline_output')
);
```

You can inspect the compiled execution plan before running anything:

```sql
SELECT ai.explain('rag_pipeline');
```

Behind the scenes, `ai.run()` translates the definition into a `pg_durable` graph. Each AI step becomes a durable node, so a failure inside `ai.embed()` does not rerun `ai.chunk()`. That is the core value: the expensive, rate-limited, occasionally unreliable model calls are checkpointed independently.

## Run, monitor, and recover

```sql
SELECT ai.run('rag_pipeline');

SELECT * FROM ai.status('rag_pipeline');
SELECT * FROM ai.list_pipelines();
```

You can drop down to the underlying durable instance for full execution history:

```sql
SELECT di.id AS instance_id, di.label, di.status AS df_status, di.created_at
FROM df.instances di
WHERE di.label = 'ai-pipeline:rag_pipeline'
ORDER BY di.created_at DESC
LIMIT 5;
```

The durable engine retries failed steps automatically, and the `azure_ai` extension handles retryable errors from the model endpoint according to the documented function behavior. Persistent failures surface in `ai.status()`. To stop the change trigger from launching new runs while you tune, use `ai.pause('rag_pipeline')` and `ai.resume('rag_pipeline')`.

## The re-embedding problem, solved as a first-class operation

Embedding models change. Dimensions change. Chunk sizes change. In the hand-rolled service-tier pattern, re-embedding a corpus is a risky one-off script. AI pipelines treat it as a built-in operation. After updating the pipeline definition or sink schema:

```sql
TRUNCATE documents_ai_pipeline_output;
SELECT ai.backfill('rag_pipeline');
```

The backfill runs as a single durable instance. If the database restarts mid-backfill, it resumes from the last checkpointed batch rather than from row zero. Combined with `incremental_column` on steady-state runs, this gives you better operational control over model usage: you embed new or changed rows on subsequent runs, and full re-embedding happens only when you explicitly ask for it.

## Where pipelines fit versus one-shot AI functions

The `azure_ai` extension also exposes one-shot model calls directly in SQL, documented in [AI functions in the azure_ai extension](https://learn.microsoft.com/en-us/azure/horizondb/ai/ai-functions): `azure_openai.create_embeddings()`, `azure_ai.generate()`, `azure_ai.extract()`, `azure_ai.is_true()`, and `azure_ai.rank()`. These are useful for interactive queries and small jobs, for example semantic search at query time:

```sql
SELECT product_name, product_description
FROM products
ORDER BY description_vector <=> azure_openai.create_embeddings(
    'my-embedding', 'Alternatives to LEGO')::vector ASC
LIMIT 10;
```

The difference is durability. A one-shot call fails the calling statement if the endpoint fails and no retry behavior is configured. A pipeline retries, resumes after a crash, and checkpoints. Use one-shot calls inside queries; use a pipeline whenever the work is large enough, long enough, or important enough that you would otherwise build a service tier for it.

## Model management

If **AI Model Management** (a limited preview) is enabled, HorizonDB provisions and registers three managed models automatically: `gpt-5.4` for chat with the alias `default-chat`, `text-embedding-3-small` for embeddings with the alias `default-embedding`, and `Cohere-rerank-v4.0-fast` for reranking with the alias `default-reranker`. There is no endpoint or key to manage for those managed models. Otherwise you deploy your own model through Microsoft Foundry and register it explicitly:

```sql
SELECT model_registry.model_add(
    'my-embedding',
    'https://YOUR_RESOURCE.openai.azure.com/',
    'text-embedding-3-small',
    'text-embedding-3-small',
    NULL,
    'subscription-key',
    'YOUR_API_KEY'
);
```

Managed identity is also a supported authentication type, which is the configuration most government security teams will prefer because it avoids storing endpoint keys in the registry.

## Why This Matters for Government

State, county, and city agencies sit on exactly the kind of unstructured corpus that RAG was built for: permit records, council minutes, benefits policy documents, case files, and 311 histories. The instinct is to grab a vector database and a separate orchestration service. For a government platform, that fragmentation is a liability:

- **Fewer moving parts to secure and authorize.** Keeping embeddings, operational data, and ingestion logic in one PostgreSQL system means one set of access controls, one audit surface, and one backup and point-in-time-recovery story instead of a vector store, a queue, and a worker fleet that each need their own hardening and authority to operate.
- **Pipeline orchestration stays close to the data.** Chunking, durable orchestration, checkpointing, sink writes, and status tracking happen inside HorizonDB. Model inference still calls a registered model endpoint, so agencies should evaluate identity, networking, logging, and data-handling requirements for that endpoint as part of the architecture.
- **Durability matches public-sector reality.** Ingestion jobs over large record sets fail partway. A pipeline that resumes from its last checkpoint, instead of a script someone has to babysit and restart, is the difference between a maintainable system and a fragile one.

Now the caveats, because they are decisive for this audience. Azure HorizonDB is in **public preview**, which means it is usually better suited to proofs of concept than production workloads under agency change-control policies. The overview currently lists commercial Azure regions only: Central US, West US 2, West US 3, Sweden Central, and Australia East. No Azure Government region is listed for the preview. Preview limitations also include service-managed keys only with no customer-managed keys yet, seven-day fixed backup retention, no cross-region read replicas, and Private Link rather than full virtual-network injection. AI pipelines themselves run only on the primary replica, and pipeline state is not portable across major `pg_durable` versions during preview.

The practical recommendation: if your organization runs workloads in commercial Azure, this is a strong candidate for a proof of concept on non-sensitive or synthetic data, specifically to validate the declarative-pipeline pattern and measure embedding cost and recall. If you are constrained to Azure Government or a GCC-aligned posture, treat this as a roadmap item, prototype against the documented API where appropriate, and track the [release notes](https://learn.microsoft.com/en-us/azure/horizondb/release-notes/release-notes) for region and compliance expansion.

## The bigger picture

The industry has spent two years treating the vector store as a separate tier bolted onto the database. Azure HorizonDB's bet, articulated in its [AI capabilities overview](https://learn.microsoft.com/en-us/azure/horizondb/ai/ai-overview), is that the database is the natural home for the whole data-to-knowledge pipeline because that is already where the data lives, where transactions commit, and where recovery is guaranteed. Declarative AI pipelines are the clearest expression of that bet so far. For government teams that have to defend every component of an architecture to a security reviewer, collapsing multiple services into one PostgreSQL-compatible engine is a compelling direction once it reaches the clouds and compliance boundaries the public sector requires.

Technical details in this post were checked against the cited Microsoft Learn documentation. Availability and preview limitations are subject to change; confirm current status in the official documentation before planning.
