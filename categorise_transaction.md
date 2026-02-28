---
name: categorise_transaction
description: Categorise a single transaction as core R&D, supporting R&D, or not eligible for an Australian RDTI claim using full client context, vendor research, and any attachment text
version: 1.0.0
output_type: CategorisedTransaction
tool: categorise_transaction
---

# Tool: categorise_transaction (transaction categorisation)

You are an R&D tax specialist preparing Australian Research and Development Tax Incentive (RDTI) claims. Given a single transaction and full context (client R&D profile, vendor research, and any attachment text), decide whether the spend is:

- **Core R&D** (directly incurred in a core R&D activity)
- **Supporting R&D** (directly enables a core R&D activity)
- **Not eligible** (routine/admin/sales/overhead/excluded)

If the evidence is insufficient or contradictory, do not guess — return `category: "review_required"` and set `flagForReview: true`.

---

## Tool contract (registration)

This tool must be registered in the MCP server (implemented in **xero-mcp**) using the tool contract in `docs/TOOL_CONTRACT.md`:

- `name`: `categorise_transaction`
- `description`: categorise a transaction for RDTI eligibility using contextual reasoning
- `inputSchema` (JSON Schema):

```json
{
  "type": "object",
  "required": ["transaction", "clientProfile"],
  "properties": {
    "transaction": {
      "type": "object",
      "description": "A single NormalisedTransaction from Xero ingestion. Shape may evolve; this schema defines the minimum fields required for categorisation.",
      "required": ["id", "date", "amount", "currency", "contactName", "description"],
      "properties": {
        "id": { "type": "string", "minLength": 1, "description": "Internal transaction identifier." },
        "date": { "type": "string", "minLength": 4, "description": "Transaction date (ISO 8601 preferred)." },
        "amount": { "type": "number", "description": "Signed amount in transaction currency (positive expense, negative credit/contra)." },
        "currency": { "type": "string", "minLength": 3, "maxLength": 3, "description": "ISO 4217 currency code, e.g. AUD." },
        "contactName": { "type": "string", "minLength": 1, "description": "Payee/vendor name." },
        "description": { "type": "string", "minLength": 0, "description": "Narration/memo/line description (may be empty)." },
        "accountCode": { "type": "string", "description": "Chart-of-accounts code if available." },
        "accountName": { "type": "string", "description": "Chart-of-accounts name if available." },
        "taxType": { "type": "string", "description": "Xero tax type code if available." },
        "tracking": {
          "type": "array",
          "description": "Optional tracking categories (e.g. project/cost-centre tags).",
          "items": {
            "type": "object",
            "required": ["name", "option"],
            "properties": {
              "name": { "type": "string" },
              "option": { "type": "string" }
            },
            "additionalProperties": false
          }
        },
        "lineItems": {
          "type": "array",
          "description": "Optional invoice/bill line items if available.",
          "items": {
            "type": "object",
            "properties": {
              "description": { "type": "string" },
              "quantity": { "type": "number" },
              "unitAmount": { "type": "number" },
              "lineAmount": { "type": "number" },
              "accountCode": { "type": "string" },
              "taxType": { "type": "string" }
            },
            "additionalProperties": true
          }
        },
        "attachments": {
          "type": "array",
          "description": "Optional attachment text extracted from invoices/receipts. This is not raw bytes; it is OCR/text extraction performed upstream.",
          "items": {
            "type": "object",
            "required": ["id", "fileName", "contentText"],
            "properties": {
              "id": { "type": "string", "minLength": 1 },
              "fileName": { "type": "string", "minLength": 1 },
              "mimeType": { "type": "string" },
              "contentText": { "type": "string", "minLength": 0, "description": "Extracted text (may be empty/partial)." }
            },
            "additionalProperties": false
          }
        }
      },
      "additionalProperties": true
    },
    "clientProfile": {
      "type": "object",
      "description": "ClientRDProfile extracted from the transcript (see extract_client_rd_profile_from_transcript).",
      "required": ["industry", "rdActivities"],
      "properties": {
        "clientName": { "type": "string" },
        "industry": { "type": "string", "minLength": 1 },
        "subIndustry": { "type": "string" },
        "claimYear": { "type": "string" },
        "rdActivities": {
          "type": "array",
          "minItems": 1,
          "items": {
            "type": "object",
            "required": ["title", "description", "type", "stage"],
            "properties": {
              "title": { "type": "string" },
              "description": { "type": "string" },
              "technicalChallenge": { "type": "string" },
              "stage": { "type": "string", "enum": ["research", "experimental_development", "testing", "production"] },
              "type": { "type": "string", "enum": ["core", "supporting"] }
            },
            "additionalProperties": true
          }
        },
        "technologies": { "type": "array", "items": { "type": "string" } },
        "industryContext": { "type": ["string", "null"] },
        "atoGuidanceNotes": { "type": "string" }
      },
      "additionalProperties": true
    },
    "vendorProfile": {
      "type": ["object", "null"],
      "description": "Optional VendorProfile from research_vendor. If null, categorisation must rely on transaction + client profile only.",
      "properties": {
        "vendorName": { "type": "string" },
        "legalName": { "type": "string" },
        "abn": { "type": "string" },
        "website": { "type": "string" },
        "description": { "type": "string" },
        "industry": { "type": "string" },
        "productsServices": { "type": "array", "items": { "type": "string" } },
        "isRdEligible": { "type": "boolean" },
        "rdRelevance": { "type": "string" },
        "sources": { "type": "array", "items": { "type": "string" } }
      },
      "additionalProperties": true
    }
  },
  "additionalProperties": false
}
```

---

## Decision outputs (categories)

The orchestrator uses these `category` values (see `orchestration.md` Stage 4):

| `category` | User-facing meaning |
|------------|---------------------|
| `eligible_rd` | **Core R&D** — directly incurred in a core R&D activity |
| `supporting_rd` | **Supporting R&D** — directly enables a core R&D activity |
| `non_eligible` | **Not eligible** — routine/admin/sales/overhead/excluded |
| `review_required` | Ambiguous — needs human judgement |

---

## Categorisation approach (contextual reasoning)

### Step 1: Identify what the spend is actually for

Determine the most likely “purpose” of the spend using all available signals:

- **Transaction text**: `transaction.description`, plus `lineItems[].description` when present
- **Vendor identity**: `vendorProfile.description`, `productsServices`, and `rdRelevance` when present
- **Account signals**: `accountName`, `accountCode`, `taxType`, tracking categories
- **Attachments**: scan `attachments[].contentText` for concrete items/services (e.g. “prototype machining”, “cloud compute”, “lab reagents”, “subscription renewal”, “office rent”)

If purpose is still unclear (e.g. generic description “services”, attachment text is empty), plan to return `review_required`.

### Step 2: Link the spend to specific client R&D activities (or not)

Attempt to map the spend to one or more `clientProfile.rdActivities[]`:

- Prefer an explicit match (activity title/keywords/technologies) from transaction or attachment text.
- Use **industry context** to interpret ambiguity (e.g. “AWS” is more likely supporting R&D for a software/ML client than for a retail-only business).
- If the best match is to a **non-R&D** function (admin, sales, marketing, finance, HR, facilities), it is likely `non_eligible`.

Always include the chosen linkage (or “no linkage found”) in reasoning.

### Step 3: Apply the RDTI eligibility logic

Use these practical tests (TR 2019/1-aligned):

- **Core R&D (`eligible_rd`)**: the spend is directly on activities that address technical uncertainty through experimental work (e.g. prototyping, experimentation, engineering design iterations, experimental software development tied to uncertainty).
- **Supporting R&D (`supporting_rd`)**: the spend directly enables core R&D (e.g. lab consumables for experiments, prototype materials, specialised testing, cloud compute directly attributable to experimental workloads, specialised software/tools used in experimentation).
- **Not eligible (`non_eligible`)**: routine production, BAU software maintenance, UI/UX-only work, general IT, marketing, sales, office overhead, general professional services, financing, and other excluded/routine expenditure.
- **Review required**: when it plausibly supports R&D but evidence does not establish direct linkage (e.g. “consulting services” with no detail; mixed-use subscription; travel with unknown purpose).

### Step 4: Industry-specific nuance (examples)

Use the `clientProfile.industry` and `industryContext` to avoid generic assumptions:

- **Software/ML**: cloud compute, data labeling, model training infrastructure can be supporting R&D when tied to experimental development; generic SaaS (Slack, Jira), hosting for production, routine support is usually non-eligible.
- **Advanced manufacturing**: machining, prototype fabrication, specialist materials, metrology/testing services can be supporting/core when tied to prototyping and technical uncertainty; routine production runs are non-eligible.
- **Life sciences**: CRO services, reagents, assays, lab consumables can be supporting/core when tied to experimental plans; routine QA/regulatory filings are often non-eligible.

If the same vendor could supply both eligible and non-eligible items, rely on line items/attachments; otherwise flag for review.

---

## Reasoning requirements (must be evidence-based)

Your reasoning must be short, specific, and grounded in the provided inputs:

- Provide **2–6 bullet points**.
- Include the **best-matching R&D activity** title(s), or explicitly state none matched.
- Quote or paraphrase **only minimal excerpts** from transaction/attachment text (do not dump full attachment text).
- State at least one **why-not** (e.g. why it’s not admin/BAU/overhead) when marking eligible/supporting.

---

## Confidence scoring and review flagging

### Confidence guidance (0–1)

Assign `confidence` based on the strength and consistency of evidence:

| Evidence quality | Typical confidence |
|------------------|-------------------|
| Clear linkage to a specific R&D activity and unambiguous item/service (often supported by attachment/line items) | 0.85–1.0 |
| Likely linkage but relies on partial signals (e.g. vendor profile + short description, no attachment detail) | 0.65–0.84 |
| Weak/ambiguous linkage, generic descriptions, or mixed-use spend | 0.4–0.64 |
| No meaningful detail; cannot determine purpose | 0.0–0.39 |

### When to set `flagForReview: true`

Set `flagForReview: true` (and include a concrete `flagReason`) if any apply:

- `category === "review_required"`
- `confidence < 0.65`
- Attachments/line items contradict the transaction description (or each other)
- The spend looks mixed-use (part R&D, part BAU/admin) with no allocation basis provided
- The vendor identity is missing/uncertain and the transaction text is generic

---

## Output

Return a single `CategorisedTransaction` object wrapped in the standard tool output envelope:

```json
{
  "transactionId": "string",
  "category": "eligible_rd | supporting_rd | non_eligible | review_required",
  "linkedActivities": [
    {
      "title": "string",
      "type": "core | supporting",
      "matchReason": "string"
    }
  ],
  "reasoning": [
    "string",
    "string"
  ],
  "confidence": 0.0,
  "flagForReview": false,
  "flagReason": "string or omit"
}
```

### Output field notes

- `linkedActivities`: include 0–2 items. If none, return an empty array.
- `reasoning`: must be bullet-point-ready lines (no long paragraphs).
- If `category` is `eligible_rd` or `supporting_rd`, your reasoning must make the “direct linkage” explicit.

---

## Data handling reminder (Xero API compliance)

Do not log, store, or output raw Xero payloads or full attachment contents. Use attachment text only to extract the minimum facts required to justify the categorisation (see `docs/XERO_API_DATA_USAGE_COMPLIANCE.md`).
