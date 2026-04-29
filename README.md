# DOCX Merge Engine

A Python-based document generation engine that populates Word (`.docx`) templates with dynamic data via a key-value API. Two functions work together: one retrieves the fields required by a template, the other merges data into it and returns the populated document.

---

## Functions

### 1. `extract_merge_fields` — Template Field Retriever

Inspects a `.docx` template and returns every merge field it contains, categorised by type.

**Input**

| Parameter | Type | Description |
|---|---|---|
| `file_contents` | `string` (base64) | The `.docx` template encoded as a base64 string |

**Output**

| Field | Type | Description |
|---|---|---|
| `count` | `integer` | Total number of unique fields found |
| `tags` | `string` | Comma-separated list of all field names |
| `files_scanned` | `string` | Which XML parts of the docx were scanned |

**Example output**
```json
{
  "count": 6,
  "tags": "#show_partner_section, #show_smsf_section, @Signature_Buyer, partner_age, partner_dob, partner_preferred_name",
  "files_scanned": "word/document.xml"
}
```

**Files scanned**

The retriever inspects the following parts of the docx zip:
- `word/document.xml` — main body
- `word/header1.xml` through `header3.xml` — headers
- `word/footer1.xml` through `footer3.xml` — footers

---

### 2. `merge_document` — Template Merge Engine

Accepts a `.docx` template and a list of key-value fields, performs all substitutions, and returns the populated document as base64.

**Input**

| Parameter | Type | Description |
|---|---|---|
| `file_contents` | `string` (base64) | The `.docx` template encoded as a base64 string |
| `fields` | `array` | List of `{ "key": "...", "value": "..." }` objects |

**Output**

| Field | Type | Description |
|---|---|---|
| `merged_docx_base64` | `string` (base64) | The populated `.docx` file |
| `unresolved_tags` | `array` | Any fields in the template that were not supplied |
| `unresolved_count` | `integer` | Count of unresolved fields |

---

## Field Types

The engine supports three field types, distinguished by prefix in the field key.

### Text fields — no prefix

Standard key-value replacement. The value is XML-escaped and inserted as plain text.

**Template**
```
{{client_preferred_name}}
```

**Payload**
```json
{ "key": "client_preferred_name", "value": "John Smith" }
```

---

### Conditional blocks — `#` prefix

Show or hide entire sections of the document based on a `true`/`false` value. The opening and closing marker tags must each sit **alone on their own line** in the Word document.

**Template**
```
{{#show_partner_section}}
Partner name: {{partner_preferred_name}}
Date of birth: {{partner_dob}}
{{/show_partner_section}}
```

**Payload — show the section**
```json
{ "key": "show_partner_section", "value": "true" }
```

**Payload — hide the section**
```json
{ "key": "show_partner_section", "value": "false" }
```

When `false` (or omitted), the entire block including all content between the markers is removed from the output document. When `true`, the markers are removed and the content is retained.

> **Template authoring rule:** `{{#block_name}}` and `{{/block_name}}` must each be on their own paragraph (their own line) in Word. They cannot share a line with other text.

---

### Image fields — `@` prefix

Fetches an image from a URL and embeds it inline into the document at the placeholder location. Supports PNG, JPEG, GIF, BMP, and WebP. PNG is recommended for signatures as it supports transparency.

**Template**
```
{{@Signature_Buyer}}
```

**Payload**
```json
{ "key": "@Signature_Buyer", "value": "https://yourbucket.s3.amazonaws.com/signatures/john-smith.png" }
```

The image is fetched at merge time, sized using its actual pixel dimensions (at 96dpi), and embedded directly into the docx. The URL must be publicly accessible or a presigned link — the engine makes a standard HTTP GET request with a 10-second timeout.

If the image URL is unreachable the field is replaced with `[Image unavailable: <reason>]` and processing continues.

> **Template authoring rule:** `{{@field_name}}` must sit alone in its own paragraph in Word, with no other text on the same line.

---

## Example API Request

```json
{
  "fields": [
    { "key": "client_preferred_name",  "value": "John Smith" },
    { "key": "soa_date",               "value": "29 April 2026" },
    { "key": "show_partner_section",   "value": "true" },
    { "key": "partner_preferred_name", "value": "Jane Smith" },
    { "key": "partner_age",            "value": "52" },
    { "key": "partner_dob",            "value": "15/06/1972" },
    { "key": "show_smsf_section",      "value": "false" },
    { "key": "@Signature_Buyer",       "value": "https://yourbucket.s3.amazonaws.com/sigs/john-smith.png" }
  ]
}
```

---

## Template Authoring Guide

### Creating fields in Word

Type the field placeholder directly into your Word document using double curly braces:

| Type | Syntax | Example |
|---|---|---|
| Text | `{{field_name}}` | `{{client_preferred_name}}` |
| Conditional open | `{{#block_name}}` | `{{#show_partner_section}}` |
| Conditional close | `{{/block_name}}` | `{{/show_partner_section}}` |
| Image | `{{@field_name}}` | `{{@Signature_Buyer}}` |

### Rules

- Field names may only contain letters, numbers, and underscores
- `{{#...}}`, `{{/...}}`, and `{{@...}}` placeholders must each be on their own line with nothing else on that line
- Text fields can appear inline within sentences, inside table cells, or anywhere in the document
- Conditional blocks can wrap any content — paragraphs, tables, multiple sections
- Nested conditional blocks are supported as long as the names are unique

### Recommended workflow

1. Build the template in Word with all `{{field_name}}` placeholders in place
2. Call `extract_merge_fields` with the template to get the full field list
3. Use the returned tags to build your data payload — `#` fields need `true`/`false`, `@` fields need URLs, plain fields need text
4. Call `merge_document` with the template and payload
5. Decode the returned `merged_docx_base64` to get the final document
6. Check `unresolved_tags` — any entries here mean the payload was missing a field the template expected

---

## Dependencies

| Library | Version | Purpose |
|---|---|---|
| `Pillow` | 11.3.0+ | Reading image dimensions for embedding |
| `requests` | standard | Fetching images from URLs |
| `zipfile` | stdlib | Reading and writing the docx zip format |
| `re` | stdlib | XML processing and field extraction |
| `base64` | stdlib | Encoding/decoding file contents |

---

## Notes

- The engine handles merge fields that Word has split across multiple XML runs (a common occurrence when typing placeholders directly in Word). The `merge_split_tags` step repairs these automatically before substitution.
- All text values are XML-escaped before injection — `&`, `<`, and `>` in values will not corrupt the document XML.
- Unresolved fields (supplied in the template but not in the payload) are left as-is in `unresolved_tags` rather than silently removed, so missing data is always visible in the response.
- The engine does not currently support table column hiding — conditional blocks operate at the paragraph level. To conditionally hide a table column, either pre-render the table content as a text field or handle column removal in a pre-processing step.
