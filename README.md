# DOCX Merge Engine

A Python-based document generation engine that populates Word (.docx) templates 
with dynamic data via a key-value API. Two functions work together: one retrieves 
the fields required by a template, the other merges data into it and returns the 
populated document.

Designed to run as a Workato Python action. Compatible with AI agents — the tag 
list returned by the retriever is self-documenting, telling the agent exactly what 
type of value each field expects.

---

## Functions

### 1. `extract_merge_fields` — Template Field Retriever

Inspects a .docx template and returns every merge field it contains, categorised 
by type.

**Input**

| Parameter | Type | Description |
|---|---|---|
| `file_contents` | string (base64) | The .docx template encoded as a base64 string |

**Output**

| Field | Type | Description |
|---|---|---|
| `count` | integer | Total number of unique fields found |
| `tags` | string | Comma-separated list of all field names |
| `tables` | array | One entry per TABLE_ROWS: table, with name, payload key, and column list |
| `files_scanned` | string | Which XML parts of the docx were scanned |

**Example output**

```json
{
  "count": 8,
  "tags": "#show_partner_section, AI:risk_profile_narrative, HTML:letter_body, IMG:adviser_signature, TABLE:portfolio_table, adviser_name, client_name, letter_date",
  "tables": [
    {
      "name": "fee_table",
      "payload_key": "TABLE_ROWS:fee_table",
      "columns": "account_owner, fee_amount, frequency"
    }
  ],
  "files_scanned": "word/document.xml"
}
```

**Files scanned**
- `word/document.xml` — main body
- `word/header1.xml` through `header3.xml` — headers
- `word/footer1.xml` through `footer3.xml` — footers

---

### 2. `merge_document` — Template Merge Engine

Accepts a .docx template and a list of key-value fields, performs all 
substitutions, and returns the populated document as base64.

**Input**

| Parameter | Type | Description |
|---|---|---|
| `file_contents` | string (base64) | The .docx template encoded as a base64 string |
| `fields` | array | List of `{ "key": "...", "value": "..." }` objects |

**Output**

| Field | Type | Description |
|---|---|---|
| `merged_docx_base64` | string (base64) | The populated .docx file |
| `unresolved_tags` | array | Fields present in the template but not supplied in the payload |
| `unresolved_count` | integer | Count of unresolved fields |

---

## Field Types

The engine supports eight field types, distinguished by prefix.

### Plain text — no prefix

Standard key-value replacement. Value is XML-escaped and inserted as plain text. 
Can appear inline within sentences, inside table cells, or anywhere in the document.

```
Dear {{client_salutation}},
Your adviser is {{adviser_name}}.
```

```json
{ "key": "client_salutation", "value": "John" },
{ "key": "adviser_name", "value": "Michael Chen" }
```

---

### Conditional blocks — `#` prefix

Show or hide sections based on a true/false value. Two usage patterns:

**Block mode** — opening and closing markers each on their own paragraph. Content 
between them is shown or hidden as a unit.
{{#show_partner_section}}
Partner name:   {{partner_name}}
Date of birth:  {{partner_dob}}
{{/show_partner_section}}

**Inline mode** — opening and closing markers within the same paragraph, wrapping 
a fragment of text.
This arrangement will be between {{client1_name}}{{#show_entity_party}} and
{{entity_name}}{{/show_entity_party}} and Intergen Advisory Partners.

When `true` the markers are removed and the content is retained. When `false` the 
markers and everything between them are removed.

```json
{ "key": "#show_partner_section", "value": "true" },
{ "key": "partner_name", "value": "Jane Smith" },
{ "key": "partner_dob", "value": "15/06/1972" }
```

> **Tip:** Conditional block names can be long and descriptive — the name itself 
> acts as an instruction to the AI agent reading the tag list:
> `{{#client_has_existing_insurance_policies}}`

---

### Row-repeating table — `TABLE_ROWS:` prefix

Populates a Word table by repeating a template row once per record. Mark the 
boundaries with `{{TABLE_START:name}}` and `{{TABLE_END:name}}` paragraphs, and 
place `{{column_name}}` tags in the template row.
{{TABLE_START:fee_table}}
[Word table — header row + template row with {{account_owner}}, {{fee_amount}}, {{frequency}}]
{{TABLE_END:fee_table}}

```json
{
  "key": "TABLE_ROWS:fee_table",
  "value": "[{\"account_owner\": \"John Smith\", \"fee_amount\": \"$1,500\", \"frequency\": \"Annually\"}]"
}
```

> **Rule:** `{{TABLE_START:name}}` and `{{TABLE_END:name}}` must each be on their 
> own paragraph, directly above and below the Word table.

> **Rule:** Column names in the payload JSON must exactly match the `{{column_name}}` 
> tags in the Word template row — case-sensitive, no spaces.

`extract_merge_fields` returns a `tables` array listing each table's name, payload 
key, and column names so the agent can construct the correct payload without 
inspecting the template manually.

---

### Image — `IMG:` prefix

Fetches an image from a URL and embeds it inline at the placeholder location. 
Supports PNG, JPEG, GIF, BMP, and WebP. PNG recommended for signatures.
{{IMG:adviser_signature}}

```json
{ "key": "IMG:adviser_signature", "value": "https://yourbucket.s3.amazonaws.com/sigs/adviser.png" }
```

> **Rule:** `{{IMG:field_name}}` must sit alone on its own line in Word.

---

### HTML table — `TABLE:` prefix

Converts an HTML table string to a native Word table. Header cells (`<th>`) receive 
bold text and a light grey background.
{{TABLE:portfolio_table}}

```json
{ "key": "TABLE:portfolio_table", "value": "<table><tr><th>Fund</th><th>Value</th></tr><tr><td>Australian Shares</td><td>$120,000</td></tr></table>" }
```

> **Rule:** `{{TABLE:field_name}}` must sit alone on its own line.

---

### Rich HTML content — `HTML:` prefix

Converts a rich HTML string to native Word XML and injects it at the placeholder 
location.

| HTML element | Word output |
|---|---|
| `<p>` | Normal paragraph |
| `<h1>` — `<h6>` | Heading 1 — Heading 6 (uses template styles) |
| `<ul>` / `<li>` | List Bullet style |
| `<ol>` / `<li>` | List Number style |
| `<strong>`, `<b>` | Bold run |
| `<em>`, `<i>` | Italic run |
| `<u>` | Underline run |
| `<br>` | Line break within paragraph |
| `<table>` | Native Word table |

A single `HTML:` field can contain any mix of paragraphs, headings, lists, and 
tables in any order.
{{HTML:letter_body}}

```json
{ "key": "HTML:letter_body", "value": "<h2>Our Recommendations</h2><p>We recommend...</p><ul><li><strong>Consolidate superannuation</strong> — merge accounts.</li></ul>" }
```

> **Rule:** `{{HTML:field_name}}` must sit alone on its own line.

> **Important — HTML vs AI prefix:** Use `HTML:` when the content is formatted 
> (paragraphs, lists, tables). Use `AI:` when the content is a short narrative 
> string that appears inline within a sentence. Unlike `HTML:`, an `AI:` field 
> can appear mid-sentence: `...because {{AI:mda_portfolio_rationale}}.`

---

### AI narrative — `AI:` prefix

Semantically identical to a plain text field — the value is inserted as-is. 
The `AI:` prefix signals to the agent that this field requires authored narrative 
content rather than a data lookup.
{{AI:executive_summary}}
{{AI:risk_profile_rationale}}

```json
{ "key": "AI:executive_summary", "value": "Based on our review of your financial position..." }
```

> **Use `HTML:` instead** when the content needs formatting (bullet lists, bold, 
> headings, tables). Use `AI:` for short plain-text narrative that sits inline 
> within existing template sentences.

---

### Page break — `{{PAGE_BREAK}}`

Inserts a Word page break. No payload entry required.

> **Rule:** `{{PAGE_BREAK}}` must sit alone on its own line.  
> **Tip:** Place `{{PAGE_BREAK}}` immediately before `{{TABLE_START:name}}` or 
> `{{TABLE:name}}` to prevent tables splitting awkwardly across pages.

---

## Field Type Summary

| Field | Payload key | Payload value | Own line required |
|---|---|---|---|
| `{{field_name}}` | `field_name` | Any string | No |
| `{{#block_name}}` / `{{/block_name}}` | `#block_name` | `true` / `false` | Block mode: yes. Inline mode: no |
| `{{TABLE_START:name}}` / `{{TABLE_END:name}}` | `TABLE_ROWS:name` | JSON array of row objects | Yes |
| `{{IMG:field_name}}` | `IMG:field_name` | Image URL | Yes |
| `{{TABLE:field_name}}` | `TABLE:field_name` | HTML table string | Yes |
| `{{HTML:field_name}}` | `HTML:field_name` | Rich HTML string | Yes |
| `{{AI:field_name}}` | `AI:field_name` | Any string (agent-authored, plain text) | No — can appear inline |
| `{{PAGE_BREAK}}` | None required | None required | Yes |

---

## Workflow

### Standard merge workflow

Read template from FileStorage (base64)
Call extract_merge_fields → get tag list + table column definitions
Build the payload (see Payload Construction below)
Call merge_document with template + payload
Decode merged_docx_base64 → save or deliver the final document
Check unresolved_tags — any entries mean the payload was missing a field


### Payload construction for AI agents

When an AI agent builds the payload from the tag list returned by 
`extract_merge_fields`, it should apply the following logic per field:

| Prefix | Agent action |
|---|---|
| `#field` | Set `true` or `false` based on scope flags / client data |
| `TABLE_ROWS:name` | Build a JSON array — column names from the `tables` output |
| `IMG:field` | Supply a publicly accessible image URL |
| `TABLE:field` | Build and supply an HTML table string |
| `HTML:field` | Author and supply a rich HTML string with lists, paragraphs, tables |
| `AI:field` | Author a plain-text narrative string based on client context |
| plain field | Look up the value from client/adviser data and supply as string |

**Handling missing data:**  
If a field value cannot be determined from available source data, output the field 
with value `{{placeholder}}` so the paraplanner can complete it manually. Never 
omit a required field entirely — omitted fields appear in `unresolved_tags` and 
leave the template tag visible in the final document.

**Conditional fields:**  
Always set all conditional (`#`) flags before authoring content fields. Content 
fields inside a false conditional block are ignored by the merge engine but should 
still be included in the payload to avoid `unresolved_tags` warnings.

---

### Reference file lookup pattern

For data that applies across many documents (adviser details, licensee config, 
product lists), store a reference file in Workato FileStorage and look it up at 
recipe run time rather than hardcoding values in the recipe.

**Example — adviser lookup:**

Read FWP_Advisers.docx from FileStorage
Find the row where adviser_name matches the SOA request
Map all columns into the payload:
adviser_name, adviser_ar_number, adviser_phone,
adviser_email, adviser_address, adviser_is_principal,
related_entity_name, licensee_name


This pattern means adviser details never need to be updated in the recipe — only 
in the reference file.

---

### Template onboarding workflow

To onboard a new template and auto-generate its field reference:

Upload the .docx template to Workato FileStorage
Call extract_merge_fields on the uploaded template
Pass the tag list + tables output to the AI agent with this prompt:
"Generate a data contract for this template. For each field, document:
the field name, prefix type, data source, expected format, and
whether it is AI-authored or a data lookup. Group by section."
Save the generated data contract as a companion .md file
alongside the template in FileStorage
Reference the data contract in the Workato recipe system prompt


This gives every template a self-describing field reference that stays in sync 
with the template itself.

---

## Template Authoring Guide

### Word styles

Heading styles in `HTML:` fields resolve against the styles defined in your 
template. Define these styles in your base template:

| Style name | Used for |
|---|---|
| Heading 1 — Heading 4 | `<h1>` — `<h4>` in HTML fields |
| List Bullet | `<ul>` / `<li>` |
| List Number | `<ol>` / `<li>` |
| Normal | `<p>` and default paragraphs |

Font, colour, and spacing are always controlled by the template styles, never 
the payload.

### Table width

All tables should be set to **AutoFit to Window** so they stretch to full page 
width on export. In Word: select table → Table Layout → AutoFit → AutoFit to Window.

To apply to all tables in a document at once, run this macro (Alt + F11 → Insert 
Module → paste → F5):

```vba
Sub FitAllTablesToWindow()
    Dim tbl As Table
    For Each tbl In ActiveDocument.Tables
        tbl.AutoFitBehavior (wdAutoFitWindow)
    Next tbl
End Sub
```

When building templates programmatically (XML/docx-js), set 
`<w:tblW w:w="0" w:type="auto"/>` on every `<w:tblPr>`.

---

## Rules Summary

- Field names may only contain letters, numbers, and underscores
- `{{IMG:...}}`, `{{TABLE:...}}`, `{{HTML:...}}`, `{{TABLE_START:...}}`, 
  `{{TABLE_END:...}}`, and `{{PAGE_BREAK}}` must each be on their own paragraph
- Plain text and `{{AI:...}}` fields can appear inline anywhere, including 
  mid-sentence
- `TABLE_ROWS:` column names in the payload JSON must exactly match 
  `{{column_name}}` tags in the Word template row — case-sensitive, no spaces
- Conditional blocks have two modes:
  - **Block mode:** `{{#name}}` and `{{/name}}` each on their own paragraph
  - **Inline mode:** `{{#name}}` and `{{/name}}` in the same paragraph
- Nested conditional blocks are supported as long as names are unique
- `{{PAGE_BREAK}}` requires no payload entry and will not appear in the tag list
- Place `{{PAGE_BREAK}}` immediately before `{{TABLE_START:name}}` or 
  `{{TABLE:name}}` to prevent page-break gaps around tables

---

## Template Examples

### Example 1 — Client Letter
{{client_name}}
{{client_address}}
{{letter_date}}
Dear {{client_salutation}},
{{HTML:letter_body}}
Yours sincerely,
{{IMG:adviser_signature}}
{{adviser_name}}
{{adviser_title}}

```json
{
  "fields": [
    { "key": "client_name",           "value": "Mr John Smith" },
    { "key": "client_address",        "value": "12 Main Street, Sydney NSW 2000" },
    { "key": "letter_date",           "value": "30 April 2026" },
    { "key": "client_salutation",     "value": "John" },
    { "key": "HTML:letter_body",      "value": "<p>Thank you for meeting with us.</p><h2>Our Recommendations</h2><ul><li><strong>Consolidate superannuation</strong> — merge three accounts into one low-cost fund.</li><li><strong>Increase salary sacrifice</strong> — contribute $27,500 p.a.</li></ul>" },
    { "key": "IMG:adviser_signature", "value": "https://yourbucket.s3.amazonaws.com/sigs/adviser.png" },
    { "key": "adviser_name",          "value": "Michael Chen" },
    { "key": "adviser_title",         "value": "Financial Adviser" }
  ]
}
```

---

### Example 2 — Statement of Advice (SOA)
{{client_name}}
{{soa_date}}
{{AI:executive_summary}}
{{PAGE_BREAK}}
{{#show_partner_section}}
Partner name:   {{partner_name}}
Date of birth:  {{partner_dob}}
{{/show_partner_section}}
{{PAGE_BREAK}}
{{TABLE_START:fee_table}}
[Word table — header row + template row with {{account_owner}}, {{fee_amount}}, {{frequency}}]
{{TABLE_END:fee_table}}
{{PAGE_BREAK}}
{{HTML:recommendations_body}}
{{PAGE_BREAK}}
{{#show_smsf_section}}
{{AI:smsf_strategy_narrative}}
{{/show_smsf_section}}
{{IMG:client_signature}}
{{client_name}}

```json
{
  "fields": [
    { "key": "client_name",               "value": "John Smith" },
    { "key": "soa_date",                  "value": "1 May 2026" },
    { "key": "AI:executive_summary",      "value": "This SOA has been prepared for John Smith following our review on 28 April 2026." },
    { "key": "#show_partner_section",     "value": "true" },
    { "key": "partner_name",              "value": "Jane Smith" },
    { "key": "partner_dob",              "value": "15/06/1972" },
    { "key": "TABLE_ROWS:fee_table",      "value": "[{\"account_owner\": \"John Smith\", \"fee_amount\": \"$1,500\", \"frequency\": \"Annually\"}]" },
    { "key": "HTML:recommendations_body", "value": "<h2>Superannuation</h2><p>We recommend consolidating your accounts.</p>" },
    { "key": "#show_smsf_section",        "value": "false" },
    { "key": "IMG:client_signature",      "value": "https://yourbucket.s3.amazonaws.com/sigs/john-smith.png" }
  ]
}
```

---

### Example 3 — Inline AI field mid-sentence

Use `AI:` when the agent authors a short phrase that sits within an existing 
template sentence rather than as a standalone block.
The recommended {{mda_portfolio_name}} Portfolio is appropriate for you
because {{AI:mda_portfolio_rationale}}.

```json
{ "key": "mda_portfolio_name",        "value": "Akambo Balanced" },
{ "key": "AI:mda_portfolio_rationale","value": "it aligns with your Growth risk profile and 15-year investment horizon" }
```

---

### Example 4 — Inline Conditionals
This arrangement will be between {{client1_name}} & {{client2_name}}{{#show_entity_party}}, and {{entity_name}}{{/show_entity_party}}, and Intergen Advisory Partners Pty Ltd.

```json
{ "key": "#show_entity_party", "value": "false" }
```

Output: `This arrangement will be between John Smith & Jane Smith, and Intergen Advisory Partners Pty Ltd.`

---

### Example 5 — Descriptive Conditional Names

Block names can be long and descriptive — they act as instructions to the agent:
{{#has_asset_allocation_variances_exceeding_10_percent}}
Asset Allocation Variance Explanation
{{HTML:asset_allocation_variance_explanation}}
{{/has_asset_allocation_variances_exceeding_10_percent}}
{{#client_has_been_declared_bankrupt_in_last_7_years}}
{{AI:bankruptcy_disclosure_narrative}}
{{/client_has_been_declared_bankrupt_in_last_7_years}}

---

### Example 6 — HTML field with inline table

```json
{ "key": "HTML:portfolio_analysis", "value": "<p>Based on your <strong>Balanced</strong> risk profile:</p><table><tr><th>Asset Class</th><th>Recommended</th></tr><tr><td>Australian Shares</td><td>35%</td></tr><tr><td>International Shares</td><td>25%</td></tr></table><p>This allocation targets <em>moderate growth</em> over <strong>15 years</strong>.</p>" }
```

---

## Dependencies

| Library | Workato version | Purpose |
|---|---|---|
| Pillow | 11.3.0 | Reading image dimensions for embedding |
| lxml | 6.0.2 | HTML parsing and Word XML generation |
| requests | 2.32.5 | Fetching images from URLs |
| zipfile | stdlib | Reading and writing the docx zip format |
| re | stdlib | XML processing and field extraction |
| base64 | stdlib | Encoding/decoding file contents |

All libraries are available natively in the Workato Python connector. No 
additional installation required.

---

## Notes

- **Split run repair** — Word sometimes splits a placeholder across multiple XML 
  runs (due to spell-checking or autocorrect). The engine detects and repairs 
  these automatically before substitution.
- **XML safety** — all plain text values are XML-escaped. `&`, `<`, and `>` in 
  values will not corrupt the document.
- **Image fallback** — if an image URL is unreachable the field is replaced with 
  `[Image unavailable: <reason>]` and the merge continues.
- **Table fallback** — if HTML table conversion fails the field is replaced with 
  `[Table error: <reason>]` and the merge continues.
- **Column hiding** — conditional blocks operate at the paragraph level. Hiding 
  an entire table column is not supported. Pass empty values or pre-render the 
  table without the column as a `TABLE:` field.
- **Unresolved tags** — fields present in the template but missing from the 
  payload appear in `unresolved_tags` rather than being silently removed.
- **Blank page removal** — when a `false` conditional block is removed, any 
  page-break paragraph immediately before it is automatically collapsed so blank 
  pages are never produced.

Key additions vs the original:

AI: vs HTML: distinction clearly explained with the inline mid-sentence example
TABLE_ROWS: column name matching rule (case-sensitive, exact match)
Payload construction logic table for AI agents
Missing data / {{placeholder}} rule
Reference file lookup pattern (adviser lookup)
Template onboarding workflow (upload → extract → AI generates data contract → save as companion .md)
AutoFit to Window guidance for tables including the XML attribute
Example 3 (inline AI field) and Example 5 updated with the real FWP conditional name
