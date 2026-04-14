---
title: Building Persistent Knowledge Bases with the LLM-Wiki Pattern and GitHub Copilot
date: 2026-04-14T14:33:39+00:00
author: Mike Hacker
tags:
- AI
- How To
- App Modernization
categories:
- AI
summary: Andrej Karpathy's LLM-Wiki pattern offers a compelling alternative to RAG by having LLMs incrementally build and maintain persistent knowledge wikis - here is how government teams can implement it today using GitHub Copilot and VS Code.
draft: false
image_prompt: A vast library interior with towering bookshelves forming a labyrinth, connected by luminous golden threads strung between book spines like a web of knowledge. A large brass compass sits open on a wooden reading table in the foreground, its needle glowing softly. Warm amber light pours through cathedral-style arched windows, casting long dramatic shadows across leather-bound volumes. The golden threads pulse gently with light where they connect books across different shelves, suggesting living interconnections. Cinematic depth of field with rich warm tones. No text, letters, numbers, or writing anywhere in the image.
image: cover.jpg
audio: audio.mp3
---

Every government IT team faces the same challenge: institutional knowledge is scattered across Teams threads, meeting notes, SharePoint libraries, policy memos, and the heads of people who may not be on the team next year. Most organizations reach for Retrieval-Augmented Generation (RAG) when they want an LLM to help, but RAG has a fundamental limitation: it rediscovers knowledge from scratch on every question. Nothing accumulates.

Andrej Karpathy, the well-known AI researcher and former head of AI at Tesla, recently published an [idea file on GitHub](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) describing a different approach he calls the **LLM-Wiki pattern**. The concept has generated significant interest in the developer community and deserves attention from government technology leaders who are looking for practical, maintainable ways to organize institutional knowledge.

## What Is the LLM-Wiki Pattern?

The core idea is straightforward but powerful. Instead of uploading documents and letting an LLM retrieve chunks at query time (the RAG approach), you have the LLM **incrementally build and maintain a persistent wiki** - a structured, interlinked collection of markdown files that sits between your team and the raw source material.

When you add a new source document, the LLM does not just index it for later retrieval. It reads the document, extracts key information, and **integrates it into the existing wiki** - updating entity pages, revising topic summaries, noting where new data contradicts old claims, and strengthening the evolving synthesis. The knowledge is compiled once and then kept current, not re-derived on every query.

As Karpathy puts it: *"The wiki is a persistent, compounding artifact. The cross-references are already there. The contradictions have already been flagged. The synthesis already reflects everything you've read."*

## Three Layers of Architecture

The pattern is elegantly simple with three layers:

### 1. Raw Sources
Your curated collection of source documents: policy memos, audit reports, vendor proposals, meeting transcripts, regulations, articles. These are immutable - the LLM reads from them but never modifies them. This is your source of truth.

### 2. The Wiki
A directory of LLM-generated markdown files: summaries, entity pages, concept pages, comparisons, an overview, a synthesis. The LLM owns this layer entirely. It creates pages, updates them when new sources arrive, maintains cross-references, and keeps everything consistent. Your team reads it; the LLM writes it.

### 3. The Schema
A configuration document that tells the LLM how the wiki is structured, what the conventions are, and what workflows to follow. In VS Code with GitHub Copilot, this maps directly to files like `.github/copilot-instructions.md` or `AGENTS.md`. This is the key configuration file that transforms a generic chatbot into a disciplined wiki maintainer.

## What Problems Does This Solve?

The LLM-Wiki pattern addresses several pain points that are especially acute in government environments:

**Knowledge loss from staff turnover.** Government agencies experience regular turnover through retirements, transfers, and election cycles. When a senior engineer or program manager leaves, their accumulated understanding of why systems were configured a certain way, what tradeoffs were made, and what the historical context was goes with them. An LLM-maintained wiki continuously captures and cross-references this knowledge.

**RAG's "cold start" problem on every query.** Traditional RAG re-derives answers from raw document chunks each time. Ask a subtle question that requires synthesizing five policy documents, and the LLM has to find and piece together the relevant fragments on every query. With the wiki pattern, that synthesis has already been performed and is continuously refined.

**Maintenance burden of traditional wikis.** The reason Confluence pages go stale and SharePoint sites become digital graveyards is not a lack of good intentions - it is that the bookkeeping cost of maintaining cross-references, keeping summaries current, and noting when new information contradicts old claims grows faster than anyone can keep up with. LLMs do not get bored and do not forget to update a cross-reference.

**Scattered institutional knowledge.** Meeting transcripts live in Teams, policy decisions live in email, technical decisions live in Jira or Azure DevOps, and vendor assessments live in SharePoint. The wiki pattern gives you a single compiled view that stays current as new sources arrive.

## Building an LLM-Wiki with GitHub Copilot in VS Code

Here is the practical part. You can start building an LLM-Wiki today using GitHub Copilot in VS Code, which is available to government organizations through [GitHub Enterprise](https://docs.github.com/en/enterprise-cloud@latest/admin/overview/about-github-enterprise-cloud). The pattern uses plain markdown files and Git, making it auditable, version-controlled, and portable.

### Step 1: Set Up the Directory Structure

Create a Git repository with the following layout:

```
knowledge-wiki/
├── .github/
│   ├── copilot-instructions.md    # The schema layer
│   ├── instructions/
│   │   └── wiki-maintenance.instructions.md
│   └── prompts/
│       ├── ingest.prompt.md
│       ├── query.prompt.md
│       └── lint.prompt.md
├── raw/                            # Raw sources (immutable)
│   ├── policies/
│   ├── meeting-notes/
│   ├── vendor-docs/
│   └── assets/
├── wiki/                           # LLM-maintained wiki
│   ├── index.md
│   ├── log.md
│   ├── entities/
│   ├── concepts/
│   ├── sources/
│   └── comparisons/
└── README.md
```

### Step 2: Define the Schema with Custom Instructions

The schema is what makes this work. Create `.github/copilot-instructions.md` with your wiki conventions. Here is an example tailored for a government IT knowledge base:

```markdown
# Wiki Maintenance Instructions

## Wiki Structure
This repository contains a knowledge wiki maintained by AI.
- `raw/` contains immutable source documents. Never modify files here.
- `wiki/` contains AI-generated markdown pages. All wiki content goes here.
- `wiki/index.md` catalogs every wiki page with a link and one-line summary.
- `wiki/log.md` is an append-only chronological record of operations.

## Page Conventions
- Use YAML frontmatter with tags, date_created, date_updated, and source_count
- Use [[wiki-links]] for cross-references between wiki pages
- Every claim should cite its source using [Source Title](../sources/source-name.md)
- Flag contradictions explicitly with a > [!WARNING] callout
- Use Mermaid diagrams for process flows and architecture

## Ingest Workflow
When processing a new source:
1. Read the source document completely
2. Create or update a summary page in wiki/sources/
3. Update wiki/index.md with the new page
4. Update all relevant entity and concept pages
5. Check for contradictions with existing wiki content
6. Append an entry to wiki/log.md with format:
   ## [YYYY-MM-DD] ingest | Source Title

## Quality Rules
- Never fabricate information not in the sources
- Preserve nuance; do not oversimplify policy positions
- Note when information may be outdated
- Track which sources support each claim
```

### Step 3: Create Reusable Prompt Files

VS Code's [prompt files](https://code.visualstudio.com/docs/copilot/customization/prompt-files) let you define reusable operations as slash commands. Create `.github/prompts/ingest.prompt.md`:

```markdown
---
agent: 'agent'
description: 'Ingest a new source document into the wiki'
tools: ['search/workspace', 'vscode/editFiles']
---
Ingest the provided source document into the knowledge wiki.

1. Read the raw source document thoroughly.
2. Discuss the 3-5 key takeaways with me before proceeding.
3. Create a source summary page in wiki/sources/.
4. Update wiki/index.md with the new entry.
5. Update or create relevant entity pages in wiki/entities/.
6. Update or create relevant concept pages in wiki/concepts/.
7. Check existing wiki pages for contradictions or updates needed.
8. Append an entry to wiki/log.md.

Follow all conventions in [copilot-instructions.md](../../.github/copilot-instructions.md).
```

Create a lint prompt at `.github/prompts/lint.prompt.md`:

```markdown
---
agent: 'agent'
description: 'Health-check the wiki for quality issues'
tools: ['search/workspace']
---
Perform a health check of the knowledge wiki. Look for:

- Contradictions between pages
- Stale claims that newer sources have superseded
- Orphan pages with no inbound links
- Important concepts mentioned but lacking their own page
- Missing cross-references
- Pages missing required YAML frontmatter

Report findings as a prioritized list and offer to fix each issue.
```

### Step 4: Start Ingesting Sources

With your structure in place, open the Chat view in VS Code (`Ctrl+Alt+I`), and start ingesting documents:

1. Drop a source file into `raw/policies/` or `raw/meeting-notes/`
2. Type `/ingest` in the Copilot Chat panel and reference the file
3. Review the key takeaways Copilot identifies
4. Let Copilot create and update wiki pages across the repository
5. Review the changes in the Git diff view
6. Commit the updates

Each source you ingest may touch 10-15 wiki pages as the LLM updates cross-references, entity pages, and the index. Over time, your wiki becomes a comprehensive, interlinked knowledge base that no human would have the patience to maintain manually.

### Step 5: Query and Expand

Ask questions against your wiki directly in Copilot Chat. Because the wiki is a set of markdown files in your workspace, Copilot can search and read them naturally:

```
Based on the wiki, what are the key differences between our 
current authentication approach and what the new state mandate 
requires? Create a comparison page if one does not exist.
```

The important insight from Karpathy's pattern: **good answers should be filed back into the wiki as new pages.** A comparison you asked for, an analysis, a connection you discovered - these compound the knowledge base just like ingested sources do.

## Scaling with Azure OpenAI

For organizations that need more control over model selection, data residency, and throughput, you can extend this pattern with [Azure OpenAI Service](https://learn.microsoft.com/en-us/azure/ai-services/openai/overview). Azure OpenAI gives you:

- **Data residency guarantees** that meet government compliance requirements
- **Private endpoints** through Azure Virtual Network integration
- **Model choice** across GPT-4o, GPT-4.1, and reasoning models like the o-series
- **Content filtering** that can be configured for your organization's policies
- **Managed identity authentication** that integrates with your existing Entra ID

For programmatic wiki operations at scale, such as batch-ingesting hundreds of policy documents, you can write a script that calls the Azure OpenAI API directly:

```python
import os
from openai import AzureOpenAI

client = AzureOpenAI(
    azure_endpoint=os.environ["AZURE_OPENAI_ENDPOINT"],
    api_version="2024-12-01-preview",
    azure_deployment="gpt-4o"
)

def ingest_source(source_path: str, wiki_dir: str):
    with open(source_path, 'r') as f:
        source_content = f.read()
    
    with open(os.path.join(wiki_dir, 'index.md'), 'r') as f:
        current_index = f.read()

    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {"role": "system", "content": open('.github/copilot-instructions.md').read()},
            {"role": "user", "content": f"""Ingest this source into the wiki.
            
Current wiki index:\n{current_index}\n\n
New source content:\n{source_content}\n\n
Return a JSON object with:\n
- summary_page: markdown content for the source summary
- updated_pages: list of {{path, content}} for pages to update
- new_pages: list of {{path, content}} for new pages to create
- index_entry: the new line to add to index.md
- log_entry: the entry for log.md"""}
        ],
        response_format={"type": "json_object"}
    )
    return response.choices[0].message.content
```

This approach lets you integrate the wiki pattern into existing CI/CD pipelines, automatically ingesting documents from SharePoint, Azure DevOps work items, or Teams meeting transcripts via [Microsoft Graph API](https://learn.microsoft.com/en-us/graph/overview).

## Why This Matters for Government

Government organizations operate under unique constraints that make the LLM-Wiki pattern particularly valuable:

**Compliance and auditability.** Because the wiki is a Git repository of markdown files, every change is tracked with full version history. You can see exactly what changed, when, and what source prompted the update. This is dramatically more auditable than a RAG system where answers are generated ephemerally.

**Staff continuity across administrations.** Government agencies face knowledge loss not just from individual turnover but from wholesale leadership changes after elections. A continuously maintained wiki preserves institutional knowledge through these transitions in a way that scattered documents cannot.

**Data sovereignty.** The entire wiki lives in files you control - in a GitHub Enterprise repository, an Azure DevOps repo, or even a local file share. Combined with Azure OpenAI's data residency guarantees, you maintain full control over sensitive policy knowledge. No data is sent to consumer AI services.

**Procurement and vendor knowledge.** Government procurement is complex, with lengthy RFPs, vendor assessments, and contract histories spanning years. An LLM-Wiki that continuously synthesizes this information helps new procurement officers get up to speed quickly and ensures historical context is not lost.

**Cross-agency collaboration.** Multiple departments often work on overlapping policy areas. A shared wiki with proper access controls can surface connections and contradictions between departmental positions that would otherwise go unnoticed.

The pattern works within existing government IT tooling: GitHub Enterprise for version control, VS Code with GitHub Copilot for the AI interaction layer, and Azure OpenAI for organizations that need dedicated model deployments.

## Getting Started

The beauty of Karpathy's approach is that it requires no special infrastructure. Here is a minimal path to getting started:

1. **Create a Git repository** with the directory structure above
2. **Install VS Code** with the [GitHub Copilot extension](https://code.visualstudio.com/docs/copilot/setup)
3. **Add your schema** as `.github/copilot-instructions.md`
4. **Drop in your first source document** and use Copilot Chat to ingest it
5. **Iterate on the schema** as you learn what conventions work for your domain

Start small - perhaps with a single project's documentation or a specific policy area. As the wiki grows and proves its value, expand the sources and refine the schema. The LLM handles the maintenance burden that traditionally kills knowledge management initiatives.

As Karpathy notes, the idea connects to Vannevar Bush's 1945 vision of the Memex - a personal, curated knowledge store with associative trails between documents. The part Bush could not solve was who does the maintenance. Today's LLMs handle that, and tools like GitHub Copilot make it accessible to any development team with a text editor and a Git repository.

## Further Reading

- [Karpathy's LLM-Wiki Gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) - The original idea file
- [VS Code Custom Instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions) - Configuring always-on AI instructions
- [VS Code Prompt Files](https://code.visualstudio.com/docs/copilot/customization/prompt-files) - Creating reusable prompt commands
- [Azure OpenAI Service Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/overview) - Enterprise LLM deployment options
- [GitHub Copilot Documentation](https://docs.github.com/en/copilot) - Getting started with Copilot
- [Microsoft Graph API](https://learn.microsoft.com/en-us/graph/overview) - Integrating with M365 data sources
