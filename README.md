# r-d_ti_ai

**Skill-definition repository** for the R&D Tax AI agent. This repo stores **.md files** that define the skills (inputs, outputs, behaviour, and tool contracts) used by the agent. The MCP server and tool implementation live in **xero-mcp**.

## Xero API data compliance (effective 2 March 2026)

Xero’s updated Developer Platform Terms prohibit using **Xero API data** to **train, fine-tune, or contribute to AI/ML model improvement** (effective **2 March 2026**). The system must treat Xero data as **ephemeral, task-scoped input** used only for classification, calculation, and claim/report generation.

See: `docs/XERO_API_DATA_USAGE_COMPLIANCE.md`.

## Contents

- **docs/PRODUCT_CONTEXT.md** — Product and Linear links, build order, tool names.
- **docs/TOOL_CONTRACT.md** — Interface all skill tools must conform to (used when implementing tools in xero-mcp).
- **docs/XERO_API_DATA_USAGE_COMPLIANCE.md** — Compliance policy for handling Xero API data (effective 2 March 2026).
- **docs/ONBOARDING.md** — Repo onboarding and implementation pointers.
- **.cursor/rules/** — Agent context for working in this repo.

Add skill-definition .md files here (e.g. per-skill docs for `analyse_transcript`, `research_vendor`, `categorise_transaction`, etc.) so the MCP server implementation in xero-mcp can follow them.

## Repositories

- **This repo (r-d_ti_ai):** Skill definitions (.md only).
- **xero-mcp:** MCP server implementation, Xero integration, tool registration (BEN-10+).
- **Product:** [Product/docs/randd-tax-ai](../Product/docs/randd-tax-ai) — product docs, architecture, work units.
