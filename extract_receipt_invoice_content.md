---
name: extract_receipt_invoice_content
description: Extract and parse text from PDF/image receipt or invoice attachments on a Xero transaction, returning both plain text and minimal structured fields for downstream categorisation
version: 1.0.0
output_type: ExtractedReceiptInvoiceContent
tool: extract_receipt_invoice_content
---

# Tool: extract_receipt_invoice_content (receipt/invoice attachment extraction)

Extract usable text content from **PDF** and **image** receipts/invoices attached to a Xero transaction, so downstream categorisation can rely on what the document actually says (line items, service descriptions, totals), not just the transaction narration.

This tool is implemented and registered in **xero-mcp**; this document defines the tool behaviour and interface.

---

## Tool contract (registration)

This tool must be registered in the MCP server (implemented in **xero-mcp**) using the tool contract in `docs/TOOL_CONTRACT.md`:

- `name`: `extract_receipt_invoice_content`
- `description`: extract + parse attachment text from PDF/image receipts/invoices for a transaction
- `inputSchema` (JSON Schema):

```json
{
  "type": "object",
  "required": ["tenantId", "transactionId"],
  "properties": {
    "tenantId": {
      "type": "string",
      "minLength": 1,
      "description": "Xero tenant ID used to retrieve attachment bytes transiently within xero-mcp."
    },
    "transactionId": {
      "type": "string",
      "minLength": 1,
      "description": "Internal transaction identifier used by the caller to correlate outputs (must match NormalisedTransaction.id)."
    },
    "attachments": {
      "type": "array",
      "description": "Optional attachment metadata if the caller already has it (recommended). If omitted, the tool may discover attachments via Xero APIs using tenantId + transactionId.",
      "items": {
        "type": "object",
        "required": ["id", "fileName"],
        "properties": {
          "id": { "type": "string", "minLength": 1, "description": "Attachment identifier (Xero attachment ID or equivalent)." },
          "fileName": { "type": "string", "minLength": 1 },
          "mimeType": { "type": "string", "description": "MIME type when known, e.g. application/pdf, image/jpeg, image/png." },
          "sizeBytes": { "type": "integer", "minimum": 0 },
          "sha256": { "type": "string", "description": "Optional hash if available; used only for deduplication within the active run. Do not persist." }
        },
        "additionalProperties": false
      }
    },
    "options": {
      "type": "object",
      "description": "Extraction tuning options. Use defaults when omitted.",
      "properties": {
        "maxCharsPerAttachment": {
          "type": "integer",
          "minimum": 1000,
          "maximum": 50000,
          "default": 12000,
          "description": "Maximum characters of extracted text to return per attachment (truncate deterministically)."
        },
        "ocrLanguageHint": {
          "type": "string",
          "default": "en",
          "description": "OCR language hint for image/scanned PDFs (BCP-47 tag preferred)."
        },
        "preferPdfTextLayer": {
          "type": "boolean",
          "default": true,
          "description": "For PDFs, attempt text-layer extraction before OCR."
        },
        "enableOcr": {
          "type": "boolean",
          "default": true,
          "description": "If true, run OCR for images and scanned PDFs when needed."
        }
      },
      "additionalProperties": false
    }
  },
  "additionalProperties": false
}
```

---

## Required behaviour

### What to extract

For each attachment (PDF or image), attempt to return:

- **`contentText`**: the plain extracted text (whitespace-normalised, truncated per `maxCharsPerAttachment`)
- **Minimal structured fields** to help categorisation without dumping full text:
  - supplier name (best effort)
  - invoice/receipt number (if present)
  - invoice/receipt date (if present)
  - currency + totals (subtotal/tax/total when present)
  - a compact list of line items (best effort; 3–15 items max)

### Extraction methods (in priority order)

- **PDFs**:
  - If a text layer exists and is non-empty, use it.
  - Otherwise treat as scanned PDF and OCR each page (within sensible limits).
- **Images**: OCR the image.

The tool must handle multi-page PDFs and rotated scans (best-effort).

### Data minimisation and redaction (required)

In outputs:

- Do not include raw bytes or base64 content.
- Do not include full addresses or bank/payment details.
- If text extraction captures likely payment identifiers (e.g. credit card numbers, BSB/account numbers), redact them before returning `contentText`.

See `docs/XERO_API_DATA_USAGE_COMPLIANCE.md` for the system-wide rules.

### Failure modes (required)

If extraction fails or the attachment is unsupported:

- Return an entry for that attachment with `extractionStatus` set accordingly.
- Set `contentText` to an empty string.
- Provide a brief `failureReason` (non-sensitive, no raw bytes).

The tool must never throw away the entire response just because one attachment fails.

---

## Output

Return a single `ExtractedReceiptInvoiceContent` object wrapped in the standard tool output envelope:

```json
{
  "transactionId": "string",
  "attachments": [
    {
      "id": "string",
      "fileName": "string",
      "mimeType": "string or omit",
      "extractionStatus": "extracted | empty | failed | unsupported",
      "contentText": "string",
      "extractedFields": {
        "supplierName": "string or omit",
        "invoiceNumber": "string or omit",
        "documentDate": "string (ISO 8601 date) or omit",
        "currency": "string (ISO 4217) or omit",
        "subtotal": "number or omit",
        "tax": "number or omit",
        "total": "number or omit",
        "lineItems": [
          {
            "description": "string",
            "quantity": "number or omit",
            "unitAmount": "number or omit",
            "lineAmount": "number or omit"
          }
        ]
      },
      "extractionConfidence": 0.0,
      "failureReason": "string or omit"
    }
  ],
  "confidence": 0.0,
  "flagForReview": false,
  "flagReason": "string or omit"
}
```

### Confidence and review flag guidance

- `extractionConfidence` is per attachment (quality of OCR/text extraction).
- Top-level `confidence` should reflect whether the set of attachments provides **usable** evidence for categorisation (e.g. at least one attachment has meaningful text).
- Set `flagForReview: true` when:
  - all attachments are `empty`, `failed`, or `unsupported`, and the caller expected attachment evidence, or
  - extracted totals appear inconsistent or contradictory across attachments.
