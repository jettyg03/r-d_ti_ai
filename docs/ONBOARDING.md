# Onboarding (Skill Definitions Repo)

This repository contains **skill-definition** documents for the R&D Tax AI agent. It does not implement any server code.

## Where code lives

- **Skill definitions (this repo)**: `.md` files describing tool behavior and orchestration.
- **MCP server + Xero integration (xero-mcp)**: tool registration, Xero API calls, secrets, persistence/logging configuration, tests.

## Compliance: Xero API data usage (effective 2 March 2026)

Xero API data must **not** be used to train, fine-tune, evaluate, or otherwise improve any AI/ML model.

When implementing tools in `xero-mcp`, ensure:

- No raw Xero payloads or attachments are persisted in long-lived storage.
- No raw payloads are logged or sent to telemetry/error reporting.
- No eval/training datasets or embeddings are built from Xero-derived prompts/outputs.

Policy reference: `docs/XERO_API_DATA_USAGE_COMPLIANCE.md`.

