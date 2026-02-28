---
name: analyse_transcript
description: Normalise transcripts (txt, docx, transcription outputs) into consistent plain text for downstream extraction
version: 3.0.0
output_type: "{ text: string }"
tool: analyse_transcript
---

# Tool: analyse_transcript (normalisation)

Normalise multiple transcript input formats (plain text, Word documents, audio transcription outputs) into a single consistent **plain text** representation for downstream extraction logic.

## When to use this tool

Use this tool whenever you receive a transcript that may be:

- raw `.txt` content (including pasted emails/chats)
- a `.docx` meeting transcript
- an audio transcription export (e.g. Whisper JSON, VTT/SRT subtitles, or other JSON-based transcript exports)

## Input

### Input shape

```json
{
  "transcript": "<raw transcript content OR base64 docx bytes OR transcription JSON>",
  "format": "auto | txt | docx_base64 | whisper_json | srt | vtt | generic_transcription_json",
  "options": {
    "removeTimestamps": true,
    "preserveSpeakerLabels": true
  }
}
```

### Format guidance

| Source | `format` | What to pass in `transcript` |
|--------|----------|------------------------------|
| Plain text | `txt` | The transcript as a string |
| Word `.docx` | `docx_base64` | Base64-encoded `.docx` bytes |
| Whisper output | `whisper_json` | The decoded JSON object (preferred) or the JSON string |
| Subtitles | `srt` / `vtt` | The file contents as a string |
| Other transcript JSON exports | `generic_transcription_json` | The decoded JSON object (preferred) or the JSON string |
| Unknown | `auto` | Best-effort detection based on content |

## Normalisation rules (required)

The tool must normalise to a single plain text output with these invariants:

- **Encoding**: output is valid UTF-8 text (no binary/control characters)
- **Newlines**: use `\n` line endings (convert CRLF/CR to LF)
- **Whitespace**: trim leading/trailing whitespace; collapse runs of spaces/tabs; collapse (>2) blank lines to at most 2
- **Ordering**: preserve the original utterance order
- **Speaker labels**: preserve explicit speaker labels when present (e.g. `Alice:`), if `options.preserveSpeakerLabels` is true
- **Timestamps**: remove timestamp/index scaffolding (SRT/VTT/typical transcription timecodes) when `options.removeTimestamps` is true, while preserving the spoken text

### Format-specific notes

- **`docx_base64`**: extract paragraph text in reading order; represent paragraph breaks as blank lines; ignore images; represent table rows as lines with cell values separated by tabs.
- **`whisper_json`**: prefer `segments[].text` concatenation (fall back to top-level `text`); do not include segment timestamps in output.
- **`srt`/`vtt`**: remove sequence numbers, time ranges, and headers like `WEBVTT`; keep only caption text lines.
- **`generic_transcription_json`**: best-effort extraction from common shapes, in priority order:
  - top-level `text` / `transcript`
  - `segments[].text`
  - `results.channels[].alternatives[].transcript`
  - `utterances[].text` (preserve diarisation labels if present)

## Output (exception to standard tool envelope)

Per `docs/TOOL_CONTRACT.md` (BEN-40), this tool returns **normalised text only**:

```json
{
  "text": "<normalised plain text transcript>"
}
```

No `confidence` or `flagForReview` fields are returned by this tool; those are the responsibility of the calling skill.
