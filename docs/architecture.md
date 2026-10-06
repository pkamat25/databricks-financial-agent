# Architecture

## Overview

A conversational QA agent that answers finance questions by routing each question to the right data source and combining the results. It handles two fundamentally different data shapes with two different mechanisms:

- **Structured data** (KPIs, transactions) → **live NL2SQL** for exact numbers
- **Unstructured data** (email, commentary) → **vector search** for context and explanation

An LLM-based router decides, per question, which tool(s) to call.

## Data layers (medallion)

| Layer | Content | Role |
|---|---|---|
| Bronze | Raw GL transactions | Drill-down detail not in reporting |
| Silver | Conformed financial facts | Standard cost-centre / account queries |
| Gold → metric view | Governed KPIs (budget/actual/forecast) | Plan-vs-actual, variance |
| Unstructured | Email + SharePoint commentary | The reason behind a variance |

## How a question flows

1. **Router** — an LLM classifies the question: does it need numbers (SQL), explanation (DOCS), or both?
2. **NL2SQL tool** — generates read-only Spark SQL against the silver table or the metric view, validates it is a SELECT, and runs it.
3. **Vector search tool** — runs semantic search over the document index to retrieve relevant commentary.
4. **Synthesis** — an LLM combines the retrieved numbers and text into one answer.

## Key decisions

**Governed semantic layer.** Structured KPIs are exposed through a Unity Catalog metric view, where measures like revenue are defined once. The agent queries these defined measures, so its numbers match official reports rather than being re-derived from raw rows — essential for finance, where figures must reconcile.

**Live SQL, not RAG, for structured data.** Retrieval returns text; it cannot reliably compute an aggregate. Finance questions need exact numbers, so structured data is queried live, not embedded and retrieved.

**RAG for unstructured data.** Explanations live in free-text email and commentary. Vector search (hybrid semantic + keyword) is the right tool for that shape.

**Metadata-driven prompt.** The schema context given to the LLM is generated from Unity Catalog column comments, not hardcoded — so documentation and the agent stay in sync.

**Safety.** The NL2SQL tool enforces read-only: any query that is not a plain SELECT is rejected.

**Reliability.** An evaluation harness runs a fixed test set and scores both source-routing and answer correctness, so prompt changes can be regression-tested.

## Scaling and production notes

- **Documentation at scale** — column descriptions are AI-generated in Unity Catalog and human-reviewed, rather than hand-written. The agent is pointed at a small curated semantic layer, not every raw table, so the documentation effort stays concentrated where it matters.
- **Unstructured ingestion** — in production, email and SharePoint content would be ingested to Delta tables via the Microsoft Graph API on a schedule. The vector index and agent are identical regardless of ingestion source; the files in `data/unstructured/` represent that ingested content.
- **Exact math at scale** — the semantic layer handles governed aggregations; for broader ad-hoc analytics, the same NL2SQL pattern extends to additional governed views.

## Tech stack

Databricks (Unity Catalog, serverless), metric views, Foundation Model APIs (Llama 3.3 70B), AI / Vector Search, Delta Lake.
