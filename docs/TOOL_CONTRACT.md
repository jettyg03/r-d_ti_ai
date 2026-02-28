# Tool registration pattern (BEN-11)

Skill-definition reference: all R&D Tax AI skill tools **implemented in xero-mcp** must conform to this interface.

## Contract

- **name** (string): Tool name, e.g. `categorise_transaction`
- **description** (string): What the tool does
- **inputSchema**: JSON Schema (or Zod shape) for validated input
- **output**: Must include:
  - Primary result (tool-specific)
  - **confidence** (0–1)
  - **flagForReview** (boolean)
  - **flagReason?** (string, optional)

> **Note (BEN-40):** `analyse_transcript` is an exception — it is a pure normalisation tool and returns `{ text: string }` only. Confidence scoring and flagging are the responsibility of the calling CoWork skill.

## Tool names used in orchestration

| Tool | Output type | Notes |
|------|-------------|-------|
| `analyse_transcript` | `{ text: string }` | Normalisation only — extraction done by CoWork skill (BEN-40) |
| `ingest_xero_data` | Transaction[] | |
| `research_vendor` | VendorProfile | |
| `categorise_transaction` | CategorisedTransaction | |
| `calculate_financials` | FinancialSummary | |
| `generate_submission` | SubmissionDocument | |

Define each skill’s behaviour and I/O in .md files in **this repo**. Implementation and registration happen in the **xero-mcp** MCP server.

## Data handling requirements (Xero API compliance — effective 2 March 2026)

All tool implementations (in **xero-mcp**) must comply with Xero’s terms effective **2 March 2026**:

- **No model improvement use**: Data obtained via the Xero API must **not** be used to train, fine-tune, evaluate, or otherwise contribute to AI/ML model improvement (including feedback loops and eval datasets).
- **No raw persistence**: Do not store raw Xero API payloads (including attachments) in long-lived storage (DB, object store, files, or durable caches). Prefer ephemeral/in-memory processing.
- **No raw logging**: Do not log raw Xero requests/responses or attachment bytes; logs must contain only non-sensitive metadata (counts, date ranges, internal IDs).

Reference policy: `docs/XERO_API_DATA_USAGE_COMPLIANCE.md`.
