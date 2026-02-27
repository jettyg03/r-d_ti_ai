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

## Tool names used in orchestration

| Tool | Output type |
|------|-------------|
| `analyse_transcript` | ClientRDProfile |
| `ingest_xero_data` | Transaction[] |
| `research_vendor` | VendorProfile |
| `categorise_transaction` | CategorisedTransaction |
| `calculate_financials` | FinancialSummary |
| `generate_submission` | SubmissionDocument |

Define each skill’s behaviour and I/O in .md files in **this repo**. Implementation and registration happen in the **xero-mcp** MCP server.
