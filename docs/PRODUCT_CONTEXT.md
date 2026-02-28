# R&D Tax AI — Product Context

This repository holds **skill-definition .md files** for the R&D Tax AI agent. The **MCP server and tool implementation** live in the **xero-mcp** repo.

## Compliance note (Xero API terms — effective 2 March 2026)

Xero API data must **not** be used to train, fine-tune, evaluate, or otherwise improve AI/ML models. Treat Xero data as **ephemeral, task-scoped input** used only for classification, calculation, and claim/report generation.

Reference: `docs/XERO_API_DATA_USAGE_COMPLIANCE.md`.

## Product documentation

- **Product docs (external):** [Product/docs/randd-tax-ai](../../Product/docs/randd-tax-ai) — overview, architecture, components, user flows.
- **Context & work units:** [Product/docs/randd-tax-ai/context](../../Product/docs/randd-tax-ai/context) — domain context, MCP contracts, and per-ticket work units (BEN-10 → BEN-39).

## Linear projects (source of truth)

| Project | Tickets | Where implemented |
|--------|---------|-------------------|
| [MCP Server & Architecture](https://linear.app/ben-g/project/randd-tax-ai-mcp-server-and-architecture-b7e9072bdeb9) | BEN-10 → BEN-14 | **xero-mcp** (server, tool contract, secrets, E2E) — except BEN-12 (orchestration skill, lives here) |
| [Xero Integration & Data Ingestion](https://linear.app/ben-g/project/randd-tax-ai-xero-integration-and-data-ingestion-05fe257979cb) | BEN-15 → BEN-19 | **xero-mcp** |
| [Vendor Research Skill](https://linear.app/ben-g/project/randd-tax-ai-vendor-research-skill-937cea358fd7) | BEN-20 → BEN-23 | Skill tool: `research_vendor` — define here, implement in xero-mcp |
| [Meeting Transcript Analyzer](https://linear.app/ben-g/project/randd-tax-ai-meeting-transcript-analyzer-d4c709c09595) | BEN-24 → BEN-27 | Skill tool: `analyse_transcript` |
| [Transaction Categorization Skill](https://linear.app/ben-g/project/randd-tax-ai-transaction-categorization-skill-3832e1f5f439) | BEN-28 → BEN-31 | Skill tool: `categorise_transaction` |
| [Financial Calculation Skill](https://linear.app/ben-g/project/randd-tax-ai-financial-calculation-skill-504be3885694) | BEN-32 → BEN-34 | Skill tool: `calculate_financials` |
| [Narrative & Submission Generator](https://linear.app/ben-g/project/randd-tax-ai-narrative-and-submission-generator-44a478ed8f55) | BEN-35 → BEN-39 | Skill tool: `generate_submission` |

## Build order (from product context)

1. MCP Server scaffold (BEN-10, BEN-11, BEN-13) — **xero-mcp**
2. Xero Integration (BEN-15 → BEN-19) — **xero-mcp**
3. Transcript Analyzer (BEN-24 → BEN-27)
4. Vendor Research (BEN-20 → BEN-23)
5. Transaction Categorisation (BEN-28 → BEN-31)
6. Financial Calculations (BEN-32 → BEN-34)
7. Narrative Generator (BEN-35 → BEN-39)
8. Orchestration skill doc (BEN-12) — **r-d_ti_ai** (agent-side; see `orchestration.md`)
9. E2E test (BEN-14) — **xero-mcp**

## Tool names (orchestration)

- `analyse_transcript` → ClientRDProfile
- `ingest_xero_data` → Transaction[] (with attachments)
- `research_vendor` → VendorProfile
- `categorise_transaction` → CategorisedTransaction
- `calculate_financials` → FinancialSummary
- `generate_submission` → SubmissionDocument

All tools must expose: `name`, `description`, `inputSchema`, and output including `confidence`, `flagForReview`, optional `flagReason`. Implement in **xero-mcp**; define behaviour and contracts in this repo (.md files).
