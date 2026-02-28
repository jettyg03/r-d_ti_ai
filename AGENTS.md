# AGENTS.md

## Cursor Cloud specific instructions

This is a **documentation-only repository** containing Markdown skill-definition files for the R&D Tax AI agent. There is no executable application, no server code, and no automated test suite. All implementation lives in the separate **xero-mcp** repository.

### What's here

- Skill-definition `.md` files (inputs, outputs, behaviour for each MCP tool)
- `docs/TOOL_CONTRACT.md` — interface contract all skill tools must follow
- `docs/PRODUCT_CONTEXT.md` — product context, Linear tickets, build order
- `.cursor/rules/` — workspace-level Cursor rules

### Lint

```sh
npm run lint        # markdownlint-cli2 on all .md files
npm run lint:fix    # auto-fix what can be fixed
```

The linter config (`.markdownlint.jsonc`) disables MD013 (line-length) and MD060 (table column style) as the existing docs use long lines and compact table pipes.

### No build / run / test

There is no application to build or run, and no test suite. The "development" workflow is editing `.md` files and linting them. The MCP server implementation and tests live in the **xero-mcp** repo.
