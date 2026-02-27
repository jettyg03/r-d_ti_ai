# r-d_ti_ai

**Skill-definition repository** for the R&D Tax AI agent. This repo stores **.md files** that define the skills (inputs, outputs, behaviour, and tool contracts) used by the agent. The MCP server and tool implementation live in **xero-mcp**.

## Contents

- **docs/PRODUCT_CONTEXT.md** — Product and Linear links, build order, tool names.
- **docs/TOOL_CONTRACT.md** — Interface all skill tools must conform to (used when implementing tools in xero-mcp).
- **.cursor/rules/** — Agent context for working in this repo.

Add skill-definition .md files here (e.g. per-skill docs for `analyse_transcript`, `research_vendor`, `categorise_transaction`, etc.) so the MCP server implementation in xero-mcp can follow them.

## Repositories

- **This repo (r-d_ti_ai):** Skill definitions (.md only).
- **xero-mcp:** MCP server implementation, Xero integration, tool registration (BEN-10+).
- **Product:** [Product/docs/randd-tax-ai](../Product/docs/randd-tax-ai) — product docs, architecture, work units.
