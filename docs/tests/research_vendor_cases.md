# Vendor research tool — test cases (BEN-23)

This document defines **deterministic acceptance test cases** for the MCP tool `research_vendor`.

- The **implementation and automated tests** live in the **xero-mcp** repo.
- These cases are designed to be executed with **stubbed** ABN Lookup and web-search responses so results do not depend on the live web.

## Case format

Each case includes:

- **Input**: the tool input passed to `research_vendor`
- **Stubs**: optional stubbed external responses (ABN Lookup, web search)
- **Expected**: assertions the automated test must enforce

## Cases

### RV-KNOWN-001 — known non-R&D supplier should be skipped

#### Input (RV-KNOWN-001)

```json
{
  "vendorName": "Officeworks"
}
```

#### Expected (RV-KNOWN-001)

- `decision === "skip"`
- `vendorProfile === null`
- `confidence >= 0.9`
- `flagForReview === false`
- `cache.hit === true`
- `cache.key === "name:officeworks"` (after normalisation)

### RV-UNKNOWN-001 — unknown vendor with no public results should be flagged

#### Input (RV-UNKNOWN-001)

```json
{
  "vendorName": "Acme Quantum Materials"
}
```

#### Stubs (RV-UNKNOWN-001)

```json
{
  "abnLookup": null,
  "webSearchResults": []
}
```

#### Expected (RV-UNKNOWN-001)

- `decision === "researched"`
- `vendorProfile.vendorName === "acme quantum materials"` (normalised)
- `vendorProfile.sources` is an empty array (no sources found)
- `confidence <= 0.29`
- `flagForReview === true`
- `flagReason` mentions that no reliable public sources were found (wording may vary)

### RV-AMBIGUOUS-001 — generic trading name with multiple plausible matches should be flagged

#### Input (RV-AMBIGUOUS-001)

```json
{
  "vendorName": "Henderson Consulting"
}
```

#### Stubs (RV-AMBIGUOUS-001)

```json
{
  "abnLookup": null,
  "webSearchResults": [
    {
      "url": "https://example-one.invalid/about",
      "title": "Henderson Consulting - About",
      "snippet": "IT consulting and managed services in Australia."
    },
    {
      "url": "https://example-two.invalid/services",
      "title": "Henderson Consulting - Services",
      "snippet": "Engineering consulting services for mining projects."
    }
  ]
}
```

#### Expected (RV-AMBIGUOUS-001)

- `decision === "researched"`
- `flagForReview === true`
- `confidence < 0.5`
- `flagReason` mentions identity ambiguity / multiple plausible companies (wording may vary)
- `vendorProfile.sources` includes at least 2 distinct domains
