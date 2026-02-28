---
name: research_vendor
description: Research a vendor (by name, optional ABN, optional website) and return a structured VendorProfile describing their business and R&D relevance for an Australian RDTI claim
version: 1.1.0
output_type: VendorProfile
tool: research_vendor
---

# Tool: research_vendor (vendor research)

You are an R&D tax specialist preparing Australian Research and Development Tax Incentive (RDTI) claims. When asked to assess an unknown or ambiguous vendor, follow these instructions carefully to decide whether research is needed and, if so, to produce a structured `VendorProfile`.

---

## Tool contract (registration)

This tool must be registered in the MCP server (implemented in **xero-mcp**) using the tool contract in `docs/TOOL_CONTRACT.md`:

- `name`: `research_vendor`
- `description`: research a vendor using public identifiers and return a `VendorProfile`
- `inputSchema` (JSON Schema):

```json
{
  "type": "object",
  "required": ["vendorName"],
  "properties": {
    "vendorName": {
      "type": "string",
      "minLength": 1,
      "description": "Vendor/payee name as provided by the caller (e.g. Xero contact name)."
    },
    "abn": {
      "type": "string",
      "description": "Optional Australian Business Number (ABN). The tool must strip non-digit characters and validate that the result is exactly 11 digits.",
      "minLength": 11,
      "maxLength": 25
    },
    "website": {
      "type": "string",
      "format": "uri",
      "description": "Optional official vendor website URL."
    }
  },
  "additionalProperties": false
}
```

---

## When to use this skill

Use this tool when the orchestrator (or another skill) encounters a payee whose business activity and R&D relevance are not already established. The tool takes **only public identifiers**:

1. **Cache/skip short-circuiting** — return a cached `VendorProfile` if fresh, or skip clearly non-R&D suppliers.
2. **Vendor profiling** — when not cached/skipped, perform public web research and return a structured `VendorProfile`.

> This document covers the ambiguity detection logic (job 1) in full and provides a skeleton for profile construction (job 2). The web-search implementation details for job 2 will be completed in BEN-21.

---

## Cache/skip short-circuit criteria

Return `decision: "use_cache"` when:

- A **fresh** cached `VendorProfile` exists for the most specific identity key available (ABN > domain > name).

Return `decision: "skip"` when:

- The vendor name unambiguously identifies a clearly non-R&D supplier (e.g. major bank, utility provider, office supplies retailer).

---

## Decision table

| Fresh cache hit? | Clearly non-R&D supplier? | Decision |
|-----------------|--------------------------|----------|
| Yes | — | `use_cache` — return cached `VendorProfile`. Stop. |
| No | Yes | `skip` — vendor is obviously non-R&D; no profile needed. |
| No | No | Proceed to Step 2 and return `decision: "researched"`. |

---

## Step 0: Check cache

Normalise the vendor name before any cache lookup:

1. Trim whitespace.
2. Convert to lowercase.
3. Remove legal suffixes: `pty ltd`, `pty. ltd.`, `ltd`, `inc`, `llc`, `co.`, `p/l`.
4. Collapse multiple spaces to one.

**Example:** `"Sigma Technologies Pty Ltd"` → `"sigma technologies"`

### Vendor research cache (BEN-22 — required)

The `research_vendor` tool must implement a caching layer so that once a vendor has been researched, the result is reused for future transactions:

- **Within a single client run**: avoid duplicate lookups across multiple transactions and minor name variations.
- **Across multiple clients**: avoid repeating web searches for the same vendor across different client datasets.

The cache must store **only public vendor research outputs** and metadata derived from public sources. It must **not** store any raw Xero API payloads, transaction descriptions, invoice references, amounts, or any client-specific context (see `docs/XERO_API_DATA_USAGE_COMPLIANCE.md`).

#### Cache identity keys (collision-resistant)

Vendors can share names, and payee strings can be ambiguous. Therefore the cache must support multiple identity keys, in this priority order:

1. **ABN key (preferred when available)**: `abn:<abnDigitsOnly>`
2. **Website domain key (when available)**: `domain:<registrableDomain>` (e.g. `domain:example.com`, `domain:example.co.uk`)
3. **Name key (fallback)**: `name:<normalisedVendorName>` (using the normalisation steps above)

The tool should store the `VendorProfile.vendorName` as the **normalised** name and may store additional `aliases` internally (e.g. raw `vendorName` variants), but aliases must not include transaction data.

#### Cache lookup rules

When handling an input vendor:

- If an **ABN** is provided, attempt lookup by `abn:<...>` first. A name-only hit must **not** override an ABN-provided request.
- Else if a **website** is provided (or discovered reliably), attempt lookup by `domain:<...>` first.
- Always attempt lookup by `name:<normalisedVendorName>` as a fallback.

Treat a cached entry as a **valid hit** only when all are true:

- The entry is **not stale** (per staleness rules below), and
- The entry does **not** have an unresolved identity ambiguity (e.g. multiple plausible companies for the same name), and
- The cached `VendorProfile` is not flagged for review with identity-related reasons.

If a cached entry exists but is **stale or ambiguous**, treat it as a **soft hit**: use it to seed/accelerate Step 2 research, but still refresh the profile before returning a final result.

#### Staleness / refresh rules

Vendor research rarely changes quickly, but low-quality sources and ambiguous identities do. Use these refresh thresholds:

- **High-quality identity** (ABN confirmed and/or official website found, confidence ≥ 0.8, not flagged): refresh after **180 days**.
- **Medium-quality identity** (website OR ABN found, confidence 0.6–0.79, not flagged): refresh after **90 days**.
- **Low-quality / ambiguous** (confidence < 0.6 or flagged): refresh after **30 days** (or always refresh when the identity is ambiguous).

When refreshing, update the cached record in place (same key) and update its `lastVerifiedAt` metadata.

#### Negative caching (skip decisions)

When the vendor is **unambiguously** a clearly non-R&D supplier (e.g. major bank, utility provider, office supplies retailer) and the tool returns `decision: "skip"` with confidence ≥ 0.9, store a negative cache entry keyed by the most specific identifier available (ABN > domain > name) with a **365 day** TTL. This avoids repeated checks for obvious non-eligible payees.

Do not negative-cache cases where the name is ambiguous or could plausibly refer to an R&D-relevant supplier.

#### Concurrency / deduplication (same run)

If the tool is called concurrently for the same cache identity key(s), it must deduplicate in-flight work so that only one web research operation executes and other calls await/reuse the same result.

### Step 0 outcome

If a **fresh** cached `VendorProfile` exists (per lookup + staleness rules), return it wrapped in the standard output envelope with `decision: "use_cache"`. Do not proceed further.

---

## Step 1: Apply cache/skip decision

Apply the decision table using only:

- The normalised `vendorName`
- Any provided `abn` and/or `website`
- The vendor research cache (Step 0)
- A small internal "clearly non-R&D supplier" list/patterns (banks, utilities, office supplies, etc.)

If the outcome is `use_cache` or `skip`, stop and return the appropriate result. Otherwise continue to Step 2.

---

## Step 2: Web search lookup _(BEN-21 — placeholder)_

Call the web search integration (configured in BEN-13) to identify what the vendor does and to gather enough evidence to classify their industry and R&D relevance.

**Data minimisation (required):**

- Only use **public identifiers** in queries: `vendorName` (normalised), optional `abn`, optional `website`.
- Do **not** include Xero transaction narration, invoice references, amounts, addresses, or any client-specific information in web queries.

### 2a. Identity confirmation (Australia-first)

1. If `abn` is provided, query **ABN Lookup** (`abr.business.gov.au`) first to confirm:
   - Legal name
   - Entity type and status
   - Registered state (if shown)
   - Any industry/description hints (if shown)
2. If `website` is provided, treat it as a strong hint but still confirm the vendor identity (common-name collisions are frequent).
3. If neither `abn` nor `website` is provided, search to find an official website and/or an authoritative registry/directory entry.

### 2b. Web search queries (use 3–6, stop early if confident)

Run a general web search with a small set of targeted queries, prioritising Australia-specific results:

- `"[vendorName] Australia"`.
- `"[vendorName] Pty Ltd"`.
- `"[vendorName] ABN"` (or `"[vendorName] ABN [abn]"` when `abn` is known).
- `"[vendorName] products services"`.
- `site:linkedin.com "[vendorName]"` (for business description corroboration).
- `site:abr.business.gov.au "[abn]"` (when `abn` is known).

### 2c. Source ranking and collection

Prefer sources in roughly this order (highest → lowest reliability):

1. **Official vendor website** (About / Products / Services pages).
2. **ABN Lookup** record (when ABN known).
3. Government / regulator / standards bodies (where relevant).
4. Reputable business directories with clear attribution (e.g. LinkedIn company page).
5. Aggregators with low verification (avoid relying on these alone).

Collect up to ~8 URLs, deduplicated by domain. Save the ordered list as `sources`. If multiple distinct companies match the name, keep sources for the top 2–3 candidates and treat this as an identity ambiguity for Step 4.

### 2d. Extract structured signals (for Step 3)

From the best available sources, extract:

- **Company description**: 1–2 factual sentences (what they do, for whom).
- **Products/services**: 3–8 specific offerings (avoid generic "consulting" unless that is genuinely all that’s stated).
- **Industry signals**: keywords/phrases indicating sector (e.g. "CRO", "precision machining", "embedded systems", "lab consumables").
- **R&D association signals** (if present): explicit mentions of engineering, prototyping, experimentation, laboratory work, scientific services, specialised manufacturing, software development, or research services.

Return these extracted signals to Step 3 (conceptually; the tool implementation should use them to populate the `VendorProfile`).

Suggested search queries:

- `"[vendorName] Australia"` — general company lookup
- `"[vendorName] ABN [abn]"` — if ABN is available
- `"[vendorName] products services R&D"`

---

## Step 3: Populate VendorProfile fields

Using the search results from Step 2, populate all fields in the `VendorProfile`:

| Field | How to populate |
|-------|----------------|
| `vendorName` | Use the normalised vendor name from Step 0. |
| `legalName` | If confidently identified (ABN Lookup or official website), record the legal entity name; otherwise omit. |
| `abn` | If provided or found reliably (ABN Lookup / official website), include; otherwise omit. |
| `website` | Official website URL if identified; otherwise omit. |
| `description` | 1–2 sentences describing what the company does. Be factual — use what the search results say. |
| `industry` | Primary industry, e.g. "Software & Technology", "Advanced Manufacturing", "Life Sciences". |
| `industryClassification` | If possible, map the vendor to an industry classification scheme. Prefer **ANZSIC 2006**: provide `scheme`, `code`, and `label`. If you cannot map confidently, set to `null` and explain uncertainty in `rdRelevance` (and potentially flag in Step 4). |
| `productsServices` | List of key products or services. Be specific — avoid generic labels like "consulting". |
| `isRdEligible` | `true` when the vendor's offerings are plausibly directly attributable to supporting/core R&D activities (e.g. engineering prototyping, CRO services, specialist materials for experiments, R&D-focused software development). Otherwise `false`. If uncertain, set `false`, explain in `rdRelevance`, and likely flag for review. |
| `rdRelevance` | Plain-English assessment of whether their offerings could constitute eligible R&D expenditure in the context of the client's RDTI claim. Reference `ClientRDProfile.industry` if available. |
| `sources` | All URLs consulted, in the order used. |

---

## Step 4: Score confidence and flag for review

### Confidence scoring

| Search result quality | Confidence |
|-----------------------|-----------|
| Official website found; all `VendorProfile` fields populated; R&D relevance is clear | 0.9 – 1.0 |
| Website found but R&D relevance is ambiguous or requires human judgement | 0.7 – 0.89 |
| Only directory listing, ABN record, or ASIC registration found | 0.5 – 0.69 |
| Vendor name only — no website, no description found | 0.3 – 0.49 |
| No useful search results returned | 0.0 – 0.29 |

Set `flagForReview: true` if **any** of the following are true:

- Confidence < 0.5
- The vendor's identity could not be confirmed (e.g. common name with multiple matching companies)
- R&D relevance is contradictory (e.g. company is both a software vendor and a facilities manager)
- The payee name appears to be a personal name with no business identity found

Include a `flagReason` describing the specific uncertainty.

---

## Output

Return the result as a `VendorProfile` JSON object:

```json
{
  "vendorName": "string — normalised",
  "legalName": "string or omit",
  "abn": "string or omit",
  "website": "https://... or omit",
  "description": "string",
  "industry": "string",
  "industryClassification": {
    "scheme": "anzsic_2006",
    "code": "string",
    "label": "string"
  },
  "productsServices": ["string"],
  "isRdEligible": true,
  "rdRelevance": "string",
  "sources": ["https://..."]
}
```

Always wrap this in the standard tool contract envelope:

```json
{
  "decision": "use_cache | researched | skip",
  "vendorProfile": { ... },
  "cache": {
    "hit": true,
    "key": "abn:<...> | domain:<...> | name:<...>",
    "stale": false,
    "scope": "run | global"
  },
  "confidence": 0.0,
  "flagForReview": false,
  "flagReason": "string or omit"
}
```

For `skip` decisions, return:

```json
{
  "vendorProfile": null,
  "decision": "skip",
  "cache": {
    "hit": true,
    "key": "name:<...>",
    "stale": false,
    "scope": "run | global"
  },
  "confidence": 1.0,
  "flagForReview": false
}
```

---

## RDTI relevance guidance

Use this as a quick reference when assessing `rdRelevance` in Step 3:

**Typically eligible** (may constitute R&D expenditure):

- Contract R&D service providers and scientific consultancies
- Specialist engineering or software development firms engaged in experimental work
- CROs (contract research organisations), universities, research institutes
- Suppliers of lab consumables, reagents, or specialist materials used in experiments
- Cloud compute providers where costs are directly attributable to R&D workloads
- Specialist equipment suppliers

**Typically not eligible** (routine or excluded expenditure):

- General professional services (legal, accounting, HR, recruitment)
- Office utilities, rent, telecommunications
- Standard SaaS subscriptions (project management, communication tools)
- Marketing, advertising, and PR agencies
- Major banks and financial institutions
- General office supplies and retail

When R&D relevance is ambiguous, describe the uncertainty in `rdRelevance` and set `flagForReview: true`.

Reference: ATO Tax Ruling TR 2019/1; AusIndustry RDTI guidance.
