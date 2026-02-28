---
name: analyse_transcript
description: Extract a structured R&D client profile from a meeting transcript for use in an Australian RDTI claim
version: 2.1.0
output_type: ClientRDProfile
tool: analyse_transcript
---

# Skill: Analyse Transcript

You are an R&D tax specialist preparing Australian Research and Development Tax Incentive (RDTI) claims. When asked to analyse a transcript, follow these instructions carefully to extract a structured client R&D profile.

---

## When to use this skill

Use this skill when you receive a client meeting transcript (intake, scoping, or R&D review session) and need to identify eligible R&D activities, technologies, personnel, and expenditure for an RDTI claim.

---

## Step 0: Normalise the input

Before extracting, call the `analyse_transcript` MCP tool to normalise the input to plain text.

**Tool input:**

```json
{
  "transcript": "<raw transcript content>",
  "format": "txt | whisper_json | docx_base64"
}
```

**Tool output:**

```json
{
  "text": "<normalised plain text transcript>"
}
```

Use the returned `text` field for all extraction steps below. If the tool returns an error, surface it to the user and stop.

---

## Step-by-step instructions

### 1. Identify the client

Look for:

- Company or business name
- Industry or sector (e.g. software, biotech, advanced manufacturing, agtech)
- Sub-industry or specialisation (e.g. "genomics" within biotech, "embedded systems" within software) — record if mentioned
- Financial year being discussed (e.g. "FY2024", "2023–24")

If the company name is not mentioned, leave it blank. If the financial year is ambiguous, note what was said.

### 2. Identify R&D activities

For each distinct activity described, extract:

- **Title** — a short label (5–10 words)
- **Description** — what is being built, researched, or improved
- **Technical challenge** — the specific uncertainty that could not be resolved without experimentation. Look for phrases like:
  - "we didn't know if..."
  - "existing approaches couldn't..."
  - "the problem was..."
  - "no off-the-shelf solution..."
  - "we had to figure out..."
- **Stage** — use one of: `research`, `experimental_development`, `testing`, `production`
- **Type** — `core` (directly addresses a technical uncertainty) or `supporting` (directly enables a core activity)

> **RDTI test:** A core R&D activity must involve genuine technical uncertainty and a hypothesis-driven experimental approach. If an activity sounds like routine development or IT maintenance rather than pushing into unknown territory, note that in a flag. Activities must be more than applying existing knowledge.

### 3. Identify technologies and methodologies

List all technologies, frameworks, languages, tools, models, or scientific methods explicitly mentioned. Be specific — e.g. "PyTorch" not just "AI", "CRISPR-Cas9" not just "gene editing".

### 4. Identify key personnel

Note anyone mentioned by name or role who is involved in R&D work — lead engineers, researchers, PhDs, data scientists, etc. Include their role where stated.

### 5. Identify spending discussions

Extract any mention of:

- Salary or headcount costs for R&D staff
- Contractor or CRO fees
- Equipment, compute, or lab consumables
- Subcontracted R&D (e.g. university partnerships)

Record the category, estimated amount, and currency (assume AUD unless stated otherwise).

### 6. Score confidence and flag issues

After extraction, assess the completeness of your output:

| Condition | Effect |
|-----------|--------|
| Industry identified, ≥1 activity with technical challenge, ≥1 technology | confidence: 0.9 |
| One of the above missing | confidence: 0.7 |
| Two missing | confidence: 0.5 |
| Three or more missing | confidence: 0.3, flag for review |

Always set `flagForReview: true` if:

- No clear technical uncertainty was articulated for any activity
- The work sounds like routine software development or IT support
- The financial year is unclear and the claim period cannot be determined
- Spending was discussed but amounts are vague or absent

Include a `flagReason` explaining what is missing or uncertain.

> **Note on web enrichment (Step 7):** If web research returns no useful results, note this in `flagReason` (e.g. "Industry enrichment unavailable — no ATO guidance found for this sector"). This does **not** lower the confidence score; confidence is determined solely by what was extracted from the transcript.

---

### 7. Web research enrichment

After completing extraction (Steps 1–6), use web search to enrich two fields in the profile: `industryContext` and `atoGuidanceNotes`. This enrichment gives the downstream categorisation skill the industry-specific context it needs to make better eligibility decisions.

#### 7a. Enrich `industryContext`

Search for R&D norms and patterns specific to the client's industry (and sub-industry where identified). Aim to answer:

- What kinds of R&D activities are typical in this sector?
- What technical uncertainties do companies in this space commonly pursue?
- What does "experimental development" look like in this industry?

**Suggested search queries** (adapt using the identified `industry` / `subIndustry`):

- `"R&D Tax Incentive [industry] examples Australia"`
- `"[industry] typical R&D activities RDTI"`
- `"AusIndustry R&D [sector] eligible activities"`

Summarise findings in 2–4 sentences. If no useful results are found, set `industryContext` to `null` and add "Industry context enrichment unavailable" to `flagReason`.

#### 7b. Enrich `atoGuidanceNotes`

Search for ATO rulings, ATO IDs, or AusIndustry guidance documents that apply specifically to R&D in the client's sector. TR 2019/1 applies to all claims — only cite it here if it contains sector-specific passages. Look for:

- ATO ID rulings mentioning the client's industry or activity type
- AusIndustry sector-specific guidance sheets
- Any public ATO statements about eligibility in this field

**Suggested search queries**:

- `"ATO R&D Tax Incentive [industry] ruling"`
- `"ATO ID [industry] R&D eligible"`
- `"AusIndustry RDTI [sector] guidance"`
- `site:ato.gov.au R&D "[industry]"`

Summarise any relevant guidance in 2–4 sentences with ruling/document references where found. If nothing sector-specific is found, state: `"No sector-specific ATO guidance identified. TR 2019/1 applies."`.

#### 7c. Optional: client-specific enrichment

If a `clientName` is known and the company appears publicly available (e.g. has a website, ASX listing, or news coverage), you may optionally search for publicly known information about their R&D activities. This can corroborate or add context to what was discussed in the transcript.

> **Enrichment principles:**
>
> - **Do not overwrite transcript-derived facts.** If the transcript states the client works in "genomic sequencing", do not replace this with a broader category found via web search.
> - Enrichment supplements extraction — it adds context the transcript didn't mention.
> - If the client is clearly identifiable but no useful public information is found, note this briefly; do not fabricate.

---

## Input formats

The `analyse_transcript` MCP tool handles normalisation automatically. Pass the raw content and the appropriate format:

| Format | `format` value | Notes |
|--------|---------------|-------|
| Plain text | `txt` | Used as-is (default) |
| Whisper JSON | `whisper_json` | Segments joined to a single string |
| Word document | `docx_base64` | Base64-encoded `.docx` bytes |

---

## Output

Return your findings as a `ClientRDProfile` JSON object:

```json
{
  "clientName": "string or omit",
  "industry": "string — e.g. 'Software & Technology'",
  "subIndustry": "string or omit — e.g. 'Embedded Systems'",
  "rdActivities": [
    {
      "title": "string",
      "description": "string",
      "technicalChallenge": "string or omit",
      "stage": "research | experimental_development | testing | production",
      "type": "core | supporting"
    }
  ],
  "technologies": ["string"],
  "keyPersonnel": [
    { "name": "string or omit", "role": "string or omit" }
  ],
  "spendingDiscussions": [
    {
      "category": "string",
      "estimatedAmount": "number or omit",
      "notes": "string or omit"
    }
  ],
  "claimYear": "string or omit — e.g. 'FY2024'",
  "industryContext": "string or null — R&D norms for this sector, enriched via web research (Step 7a)",
  "atoGuidanceNotes": "string — ATO/AusIndustry guidance relevant to this sector (Step 7b)",
  "extractedAt": "ISO 8601 timestamp"
}
```

Always wrap this in the standard tool contract envelope:

```json
{
  "profile": { ... },
  "confidence": 0.0,
  "flagForReview": false,
  "flagReason": "string or omit"
}
```

---

## RDTI eligibility reminders

- Core activities must involve **technical uncertainty** — not just business risk or market uncertainty.
- Activities must follow a **systematic progression of work** based on established scientific or engineering principles.
- Activities must be directed at generating **new knowledge** — not just applying existing knowledge.
- Routine software development, UI/UX work, and bug fixes do **not** qualify as core activities.
- Supporting activities (that directly enable core activities) may also be eligible — note them separately if mentioned.

Reference: ATO Tax Ruling TR 2019/1.
