# Xero API Data Usage Compliance (Effective 2 March 2026)

Beginning **2 March 2026**, Xero’s Developer Platform Terms prohibit using data obtained through the **Xero API** to **train**, **fine-tune**, or otherwise **contribute to AI/ML model improvement**. This is a **hard compliance requirement** for the R&D Tax AI system.

This repository (`r-d_ti_ai`) contains **skill-definition** documents only. The MCP server and Xero integration are implemented in **xero-mcp**. This document defines the **required data-handling behavior** for the full pipeline and must be reflected in the xero-mcp implementation.

## Scope and definitions

- **Xero Data**: Any data retrieved via the Xero API, including (but not limited to) transactions, contacts, account codes, invoices/bills, line items, descriptions, tracking categories, and **attachments** (e.g. invoices/receipts).
- **Permitted processing**: Using Xero Data strictly to classify/categorise transactions, compute totals, and generate the R&D Tax Incentive claim outputs for the specific client and claim year requested.
- **Prohibited model use**: Any use of Xero Data to improve an AI/ML model, including training, fine-tuning, RLHF-style feedback loops, building eval datasets, prompt-tuning datasets, embeddings/vector stores intended for reuse, or “telemetry” corpora.

## Data flow audit (intended architecture)

### Where Xero Data enters

- **Stage 2** in `orchestration.md`: `ingest_xero_data` fetches Xero transactions (optionally including attachments).

### How Xero Data is used downstream

- **Stage 3** vendor research uses **vendor names** (`contactName`) as lookups. Vendor profiles should be derived from public web sources and **must not require storing Xero transaction payloads**.
- **Stage 4** categorisation uses transaction fields (date, amount, account, description) to assign eligibility categories.
- **Stage 5** calculation aggregates categorised transactions into totals and RDTI estimates.
- **Stage 6** submission generation uses the client profile plus financial summaries and (where necessary) minimal transaction detail to produce schedules and narratives.

### Where Xero Data must NOT go

- Any **model training / fine-tuning** pipeline, data lake, analytics warehouse, vector database, or evaluation dataset store.
- Any “improvement” feedback loop that stores Xero-derived inputs/outputs for later training/evaluation.
- Application logs, request/response logs, APM traces, error reporting payloads, or debugging dumps containing raw Xero payloads.

## Required handling rules

### Permitted uses (allowed)

- **On-demand** retrieval and processing of Xero Data for the specific tenant and claim year.
- Classification/categorisation, aggregation, and report/claim document generation for the requesting user.
- Displaying **summaries** (counts, totals, category breakdowns) and **minimal transaction excerpts** when necessary for human review.

### Prohibited uses (not allowed)

- Using Xero Data to train, fine-tune, evaluate, benchmark, or otherwise improve any AI/ML model.
- Persisting Xero Data in any reusable dataset intended for evaluation, labeling, or prompt library creation.
- Generating embeddings from Xero Data for storage in a vector DB for reuse beyond the immediate user task.
- Exporting or sharing Xero Data with third parties except as required to deliver the client’s outputs (e.g. user-approved report export).

## Retention and storage policy (system-wide)

- **Default retention**: **ephemeral only**. Xero Data should be held in memory only for the duration of the active pipeline run.
- **No central persistence**: Do not store raw Xero payloads (including attachments) in server databases, object stores, caches, or long-lived files.
- **User outputs only**: Persist only the *minimum* necessary derived outputs (e.g. aggregated totals, category summaries, submission document) and avoid embedding raw Xero transaction payloads unless required for schedules the user explicitly requests.
- **Attachments**: Treat as highly sensitive. If `includeAttachments` is enabled, do not persist attachment bytes; process transiently and prefer extracting only the minimum fields needed for categorisation/review.

## Logging and telemetry policy

- **No raw payload logs**: Never log full Xero API responses, transaction arrays, or attachment data.
- **Redaction**: If logging is unavoidable for debugging, log only non-sensitive metadata (counts, date ranges, internal IDs) and ensure PII and free-text descriptions are excluded.
- **Error reporting**: Error payloads must not include raw request/response bodies from Xero.

## Engineering checklist (for xero-mcp implementers)

- Ensure `ingest_xero_data` does **not** write raw responses to disk, DB, or object storage.
- Disable/avoid any HTTP middleware that logs full request/response bodies for Xero routes.
- Ensure tool outputs returned to the agent are the **minimum necessary** to complete the workflow.
- Do not create eval/training corpora from any Xero-derived prompts, completions, labels, rationales, or attachments.
- If a cache is introduced for performance, it must be **short-lived**, **encrypted**, and store only **non-sensitive** derived aggregates (not raw payloads) — and still must not be used for model improvement.

## Documentation requirement

- This compliance note must also appear in the **xero-mcp** README/onboarding docs and any internal architecture docs describing the pipeline.
