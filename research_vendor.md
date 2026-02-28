---
name: research_vendor
description: Determine whether a vendor requires active research and, if so, build a structured VendorProfile describing their business and R&D relevance for an Australian RDTI claim
version: 1.1.0
output_type: VendorProfile
tool: research_vendor
---

# Skill: Research Vendor

You are an R&D tax specialist preparing Australian Research and Development Tax Incentive (RDTI) claims. When asked to assess an unknown or ambiguous vendor, follow these instructions carefully to decide whether research is needed and, if so, to produce a structured `VendorProfile`.

---

## When to use this skill

Use this skill when the orchestrator encounters a payee in Xero transaction data whose R&D relevance is not already established. The skill has two jobs:

1. **Ambiguity detection** — decide whether active research is required, or whether the vendor can be resolved from cache or skipped as obviously non-eligible.
2. **Profile construction** — if research is required, gather information and populate a `VendorProfile`.

> This document covers the ambiguity detection logic (job 1) in full and provides a skeleton for profile construction (job 2). The web-search implementation details for job 2 will be completed in BEN-21.

---

## Ambiguity detection criteria

Flag a vendor for research (`needs_research`) when **any** of the following conditions is true:

| # | Criterion | Detail |
|---|-----------|--------|
| 1 | **Unknown vendor name** | The normalised payee name does not appear in the vendor cache or the known-supplier list. |
| 2 | **Vague transaction description** | The Xero transaction narration/reference contains only generic terms — e.g. "consulting", "services", "materials", "fees", "invoice" — or is blank. |
| 3 | **No prior categorisation history** | No previously categorised transaction exists for this payee in the current client run or cross-client history. |
| 4 | **Name does not obviously map to a known category** | The payee name gives no clear signal about the type of goods or services supplied (e.g. "Henderson Consulting", "Sigma Technologies Pty Ltd", "Blue River Holdings"). |

**Skip** the vendor (no research needed) when **either** of the following is true:

- A cached `VendorProfile` already exists for the normalised vendor name — return it directly (see Step 0).
- The payee name unambiguously identifies a clearly non-R&D supplier — e.g. a utility provider, major bank, or obvious retail/admin supplier (e.g. "AGL Energy", "Commonwealth Bank", "Officeworks").

---

## Ambiguity decision table

| Cached? | Description vague? | Name maps to known category? | Prior categorisation? | Decision |
|---------|--------------------|------------------------------|-----------------------|----------|
| Yes | — | — | — | `use_cache` — return cached `VendorProfile`. Stop. |
| No | No | Yes (clearly non-R&D) | — | `skip` — vendor is obviously ineligible; no profile needed. |
| No | No | Yes (non-ambiguous, R&D-possible) | Yes | `use_cache` — reuse prior categorisation context; no new web search. |
| No | No | Yes (non-ambiguous, R&D-possible) | No | `needs_research` — name is R&D-possible but no history; research to establish profile. |
| No | Yes | Any | Any | `needs_research` — proceed to Steps 1–4. |
| No | No | No | No | `needs_research` — name is ambiguous; proceed to Steps 1–4. |
| No | No | No | Yes | `needs_research` — prior categorisation exists but name is still ambiguous; research to confirm. |

---

## Step 0: Check cache

Normalise the vendor name before any cache lookup:

1. Trim whitespace.
2. Convert to lowercase.
3. Remove legal suffixes: `pty ltd`, `pty. ltd.`, `ltd`, `inc`, `llc`, `co.`, `p/l`.
4. Collapse multiple spaces to one.

**Example:** `"Sigma Technologies Pty Ltd"` → `"sigma technologies"`

If a cached `VendorProfile` exists for the normalised name, return it wrapped in the standard output envelope. Do not proceed further.

---

## Step 1: Apply ambiguity detection

Run the 4-criteria check against:

- The normalised payee name
- The Xero transaction description/narration
- The client's prior categorisation history
- The known-supplier list (if configured)

Determine the decision outcome using the decision table above.

- `use_cache` or `skip` → stop and return the appropriate result.
- `needs_research` → continue to Step 2.

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
  "vendorProfile": { ... },
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
