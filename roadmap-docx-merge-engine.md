# DOCX Merge Engine — Roadmap

---

## Shipped

| Feature | Description |
|---|---|
| Plain text substitution | `{{field_name}}` anywhere in the document |
| Conditional blocks | `{{#block}}` / `{{/block}}` — block and inline modes |
| Row-repeating tables | `TABLE_ROWS:` — one row per record, columns fixed in template |
| Column-repeating tables | `TABLE_COLS:` — one column per record, row labels fixed in template |
| Repeating sections | `REPEAT:` — stamp out rich document blocks once per record, full field type support inside |
| Image embedding | `IMG:` — fetch from URL, embed inline. `__Xcm` width hint for sizing |
| Rich HTML content | `HTML:` — paragraphs, headings, lists, tables converted to native Word XML |
| AI narrative fields | `AI:` — signals agent to author narrative; works inline mid-sentence |
| Page breaks | `{{PAGE_BREAK}}` — inserts Word page break |
| Split-run repair | Auto-repairs placeholders Word has split across XML runs |
| Blank page removal | Collapses orphaned page breaks after false conditional blocks |
| Extract merge fields | Self-describing tag discovery — returns field keys, table schemas, repeat block schemas |
| HTML list rendering | `<ul>` / `<ol>` with proper Word numbering definitions |

---

## Roadmap

### Priority 1 — High impact, relatively contained

#### PDF output
Convert the merged `.docx` to PDF and return both files (or PDF only, configurable).

Almost every real delivery workflow needs PDF — advisers send PDFs not Word docs. Currently this requires a separate conversion step outside the engine.

**Approach:** Call a conversion API (e.g. ConvertAPI, LibreOffice headless) after merge. Add `output_format` input field (`docx` | `pdf` | `both`). Return `merged_pdf_base64` alongside the existing `merged_docx_base64`.

**Input change:** Add optional `output_format` field (defaults to `docx` for backward compatibility).  
**Output change:** Add optional `merged_pdf_base64` field.

---

#### Number and date formatting hints
Allow template authors to declare how a raw value should be formatted inline in the tag, so AI agents supply raw data and the engine handles presentation.

```
{{amount|currency}}        →  $145,000
{{rate|percent}}           →  0.68%
{{date|long}}              →  18 May 2026
{{date|short}}             →  18/05/2026
{{amount|currency_nodec}}  →  $145,000 (no cents)
```

**Approach:** Extend the plain text substitution step to parse `|format` hints, apply formatting, and strip the hint before output. No payload change — agents supply raw values.

**Impact:** Eliminates formatting responsibility from AI agents; ensures consistent number presentation across all documents from the same data source.

---

### Priority 2 — Medium impact

#### Multi-template assembly
Accept an array of templates and merge them end-to-end into a single output document. Each template section can have its own fields.

Useful for SOAs assembled from separate section templates (cover page, recommendations, fee schedule, disclosure) that are maintained independently.

```json
{
  "templates": [
    { "file_contents": "base64...", "fields": [...] },
    { "file_contents": "base64...", "fields": [...] }
  ]
}
```

**Approach:** Merge each template independently, then concatenate the document body XML with a page break between sections, preserving each template's styles.

---

#### Nested REPEAT blocks
Allow a `REPEAT:` block inside another `REPEAT:` block, e.g. multiple properties per client, multiple line items per recommendation.

```
{{REPEAT_START:clients}}
  {{client_name}}
  {{REPEAT_START:properties}}
    {{property_address}}
    {{property_value}}
  {{REPEAT_END:properties}}
{{REPEAT_END:clients}}
```

**Approach:** Process inner repeat blocks first (innermost-first traversal) before processing outer blocks. Requires scoped field resolution so inner records inherit outer record fields.

---

#### Conditional column hiding in TABLE_COLS
Allow individual columns in a `TABLE_COLS:` table to be hidden based on a flag, without having to rebuild the entire table.

Currently the only workaround is to pass empty values or pre-filter columns in the payload.

**Approach:** Add an optional `hidden_columns` key to the `TABLE_COLS:` payload, listing column key names to exclude from rendering.

---

### Priority 3 — Lower priority / niche

#### Hyperlinks
Inject clickable hyperlinks into the document.

```
{{LINK:text|https://example.com}}
```

**Approach:** Add `LINK:` as a new field type. Inject as a Word hyperlink relationship + `<w:hyperlink>` XML element.

---

#### Tracked changes / comments
Insert reviewer comments or tracked change markup programmatically. Useful for compliance workflows where specific disclosures must be flagged for adviser review.

---

#### Table of contents generation
Auto-generate a Word TOC based on Heading styles in the merged document. Useful for long SOAs.

---

## Design principles

These principles guide all roadmap decisions:

- **Template is the schema** — the template author defines structure and styling in Word; the engine just fills data. No configuration outside the template itself.
- **Agent-friendly** — `extract_merge_fields` returns everything an AI agent needs to build a correct payload without inspecting the template.
- **Backward compatible** — new features never break existing templates or payloads. New input fields are optional with safe defaults.
- **Fail gracefully** — unresolved tags appear in `unresolved_tags`; image/table errors produce inline fallback text; the merge always completes.
