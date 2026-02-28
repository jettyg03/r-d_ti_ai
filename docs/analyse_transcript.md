# analyse_transcript — developer reference

Tool definition (normalisation-only): [`/analyse_transcript.md`](../analyse_transcript.md)

CoWork skill (extraction + enrichment): [`/extract_client_rd_profile_from_transcript.md`](../extract_client_rd_profile_from_transcript.md)

Implementation: [`xero-mcp/src/transcript/`](../../xero-mcp/src/transcript/)

Linear: BEN-24 (extraction), BEN-25 (enrichment), BEN-26 (multi-format), BEN-27 (MCP registration + tests), BEN-40 (slim to normalisation only)

## Architecture change (BEN-40)

The MCP tool now handles **normalisation only** — it accepts `txt`, `whisper_json`, or `docx_base64` input and returns plain text. The LLM-based extraction and enrichment steps have been removed from the tool and moved to this CoWork skill document.

**Before (BEN-27):** Claude CoWork → `analyse_transcript` MCP → Claude (extract) → Claude (enrich) → `ClientRDProfile`

**After (BEN-40):** Claude CoWork → `analyse_transcript` MCP (normalise only) → Claude CoWork extracts `ClientRDProfile` per skill doc
