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
| `tables` | array | One entry per TABLE_ROWS table, with name, payload key, and column list |
| `col_tables` | array | One entry per TABLE_COLS table, with name, payload key, and row key list |
| `files_scanned` | string | Which XML parts of the docx were scanned |

**Example output**

```json
{
  "count": 7,
  "tags": "#show_partner_section, AI:risk_profile_narrative, HTML:letter_body, IMG:adviser_signature, adviser_name, client_name, letter_date",
  "tables": [
    {
      "name": "fee_table",
      "type": "TABLE_ROWS",
      "payload_key": "TABLE_ROWS:fee_table",
      "columns": "account_owner, fee_amount, frequency"
    }
  ],
  "col_tables": [
    {
      "name": "current_funds",
      "type": "TABLE_COLS",
      "payload_key": "TABLE_COLS:current_funds",
      "row_keys": "balance, investment_fee, orr_levy, product, sliding_admin_fee, total_combined_costs, total_product_costs"
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
```
{{#show_partner_section}}
Partner name:   {{partner_name}}
Date of birth:  {{partner_dob}}
{{/show_partner_section}}
```

**Inline mode** — opening and closing markers within the same paragraph, wrapping 
a fragment of text.
```
This arrangement will be between {{client1_name}}{{#show_entity_party}} and
{{entity_name}}{{/show_entity_party}} and Intergen Advisory Partners.
```

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

```
{{TABLE_START:fee_table}}
[Word table — header row + template row with {{account_owner}}, {{fee_amount}}, {{frequency}}]
{{TABLE_END:fee_table}}
```

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

### Column-repeating table — `TABLE_COLS:` prefix

Populates a Word table where the **rows are fixed** and the **columns are dynamic** 
— one column per record. The classic use case is a product comparison table: 
row labels are fixed in the template (Product, Balance, Investment fee, etc.) and 
each fund or option becomes a new column at runtime.

**How to author the template in Word:**

Build the table exactly as it should look, with all row labels, subheaders, 
indentation, bold rows, and shading defined in column 1. In column 2 — and only 
column 2 — place `{{TABLE_COLS:name}}` in the header cell and a `{{placeholder}}` 
in each data row. Leave subheader rows, spacer rows, and total rows empty in 
column 2; the engine will clone the empty cell (preserving its styling) once per 
record. No wrapper markers needed.

```
| Fee                  | {{TABLE_COLS:current_funds}} |
|----------------------|------------------------------|
| Product              | {{product}}                  |
| Balance              | {{balance}}                  |
| Ongoing fees         |                              |  ← subheader, empty cell
|   Investment fee     | {{investment_fee}}           |
|   Sliding admin fee  | {{sliding_admin_fee}}        |
|   Admin fee (flat)   | {{admin_fee_flat}}           |
|   ORR levy           | {{orr_levy}}                 |
| Total product costs  | {{total_product_costs}}      |
| Total combined costs | {{total_combined_costs}}     |
```

> **Rule:** All `{{placeholders}}` must appear only in the last (rightmost) column. 
> Column 1 is never modified by the engine.

> **Rule:** Placeholder names can be anything — `{{product}}`, `{{col_product}}`, 
> `{{fund_product}}` are all valid. The name just needs to match the key in the payload.

> **Rule:** Leave subheader, spacer, and section-break cells empty in the template 
> column. The engine clones them as-is, preserving shading and borders, with no text.

The payload is an array of objects — one object per column to generate:

```json
{
  "key": "TABLE_COLS:current_funds",
  "value": "[{\"product\": \"AMP MySuper\", \"balance\": \"$145,000\", \"investment_fee\": \"0.68%\", \"sliding_admin_fee\": \"$185 p.a.\", \"admin_fee_flat\": \"$52 p.a.\", \"orr_levy\": \"$14.50\", \"total_product_costs\": \"$1,233\", \"total_combined_costs\": \"$1,233\"}, {\"product\": \"REST Core Strategy\", \"balance\": \"$62,000\", \"investment_fee\": \"0.55%\", \"sliding_admin_fee\": \"$78 p.a.\", \"admin_fee_flat\": \"$0\", \"orr_levy\": \"$6.20\", \"total_product_costs\": \"$425\", \"total_combined_costs\": \"$425\"}]"
}
```

`extract_merge_fields` returns a `col_tables` array with the table name, payload 
key, and `row_keys` — the placeholder names the agent needs to populate per record.

**TABLE_ROWS vs TABLE_COLS — which to use:**

| | TABLE_ROWS | TABLE_COLS |
|---|---|---|
| Fixed axis | Columns (headers) | Rows (labels) |
| Dynamic axis | Rows (one per record) | Columns (one per record) |
| Typical use | Fee schedules, transaction lists | Product comparisons, fund comparisons |
| Wrapper markers needed | Yes — TABLE_START / TABLE_END | No |

---

### Image — `IMG:` prefix

Fetches an image from a URL and embeds it inline at the placeholder location. 
Supports PNG, JPEG, GIF, BMP, and WebP. PNG recommended for signatures.

**Width hint — `__Xcm` suffix**

Append `__Xcm` to the tag name to set the rendered width. Height is always derived 
proportionally from the image's natural aspect ratio. The hint is encoded in the 
template tag, not the payload — the agent always supplies just the URL.

```
{{IMG:adviser_signature__3cm}}
{{IMG:graph__15cm}}
{{IMG:client_photo__5cm}}
```

If no hint is supplied the engine caps the image at 15.4 cm wide × 8 cm tall 
(proportional), which suits most charts. For small images like signatures, always 
supply a hint.

**Recommended widths by image type:**

| Image type | Recommended tag |
|---|---|
| Adviser / client signature | `__3cm` |
| Chart / graph | `__15cm` |
| Photo | `__5cm` |
| Half-width diagram | `__8cm` |

`extract_merge_fields` returns the canonical tag name with the hint stripped — 
e.g. `IMG:graph` — so the agent never needs to know about sizing. It just supplies 
the URL.

```json
{ "key": "IMG:adviser_signature", "value": "https://yourbucket.s3.amazonaws.com/sigs/adviser.png" },
{ "key": "IMG:graph",             "value": "https://quickchart.io/chart?c=..." }
```

> **Rule:** `{{IMG:field_name__Xcm}}` must sit alone on its own line in Word.

---

### Rich HTML content — `HTML:` prefix

Converts a rich HTML string to native Word XML and injects it at the placeholder 
location. Use for any formatted content — paragraphs, headings, lists, and tables 
can all be passed in a single field.

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

```
{{HTML:letter_body}}
```

```json
{ "key": "HTML:letter_body", "value": "<h2>Our Recommendations</h2><p>We recommend...</p><ul><li><strong>Consolidate superannuation</strong> — merge accounts.</li></ul>" }
```

> **Rule:** `{{HTML:field_name}}` must sit alone on its own line.

> **HTML vs AI:** Use `HTML:` when the content is formatted (paragraphs, lists, 
> tables). Use `AI:` when the content is a short plain-text narrative that sits 
> inline within an existing template sentence.

---

### AI narrative — `AI:` prefix

Semantically identical to a plain text field — the value is inserted as-is. 
The `AI:` prefix signals to the agent that this field requires authored narrative 
content rather than a data lookup. Unlike `HTML:`, an `AI:` field can appear 
mid-sentence.

```
{{AI:executive_summary}}

...because {{AI:mda_portfolio_rationale}}.
```

```json
{ "key": "AI:executive_summary", "value": "Based on our review of your financial position..." },
{ "key": "AI:mda_portfolio_rationale", "value": "it aligns with your Growth risk profile and 15-year investment horizon" }
```

> **Use `HTML:` instead** when the content needs formatting (bullet lists, bold, 
> headings, tables).

---

### Page break — `{{PAGE_BREAK}}`

Inserts a Word page break. No payload entry required.

> **Rule:** `{{PAGE_BREAK}}` must sit alone on its own line.  
> **Tip:** Place `{{PAGE_BREAK}}` immediately before `{{TABLE_START:name}}` to 
> prevent tables splitting awkwardly across pages.

---

## Field Type Summary

| Field | Payload key | Payload value | Own line required |
|---|---|---|---|
| `{{field_name}}` | `field_name` | Any string | No |
| `{{#block_name}}` / `{{/block_name}}` | `#block_name` | `true` / `false` | Block mode: yes. Inline mode: no |
| `{{TABLE_START:name}}` / `{{TABLE_END:name}}` | `TABLE_ROWS:name` | JSON array of row objects | Yes |
| `{{TABLE_COLS:name}}` in table header cell | `TABLE_COLS:name` | JSON array of column objects | No — marker lives inside the table |
| `{{IMG:field_name__Xcm}}` | `IMG:field_name` | Image URL | Yes — hint stripped from payload key |
| `{{HTML:field_name}}` | `HTML:field_name` | Rich HTML string (paragraphs, lists, tables) | Yes |
| `{{AI:field_name}}` | `AI:field_name` | Plain-text narrative string | No — can appear inline |
| `{{PAGE_BREAK}}` | None required | None required | Yes |

> **Note — TABLE_ROWS vs TABLE_COLS:** `TABLE_ROWS` tables are wrapped in 
> `{{TABLE_START:name}}` / `{{TABLE_END:name}}` marker paragraphs. `TABLE_COLS` 
> tables need no markers — the `{{TABLE_COLS:name}}` tag inside the header cell 
> of the template column is sufficient for both authoring and detection.

---

## Workflow

### Standard merge workflow

1. Read template from FileStorage (base64)
2. Call `extract_merge_fields` → get tag list + table definitions
3. Build the payload (see Payload Construction below)
4. Call `merge_document` with template + payload
5. Decode `merged_docx_base64` → save or deliver the final document
6. Check `unresolved_tags` — any entries mean the payload was missing a field

### Payload construction for AI agents

When an AI agent builds the payload from the tag list returned by 
`extract_merge_fields`, it should apply the following logic per field:

| Prefix | Agent action |
|---|---|
| `#field` | Set `true` or `false` based on scope flags / client data |
| `TABLE_ROWS:name` | Build a JSON array of row objects — key names from `tables[].columns` |
| `TABLE_COLS:name` | Build a JSON array of column objects — key names from `col_tables[].row_keys` |
| `IMG:field` | Supply a publicly accessible image URL |
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

1. Read `FWP_Advisers.docx` from FileStorage
2. Find the row where `adviser_name` matches the SOA request
3. Map all columns into the payload:
   `adviser_name`, `adviser_ar_number`, `adviser_phone`,
   `adviser_email`, `adviser_address`, `adviser_is_principal`,
   `related_entity_name`, `licensee_name`

This pattern means adviser details never need to be updated in the recipe — only 
in the reference file.

---

### Template onboarding workflow

To onboard a new template and auto-generate its field reference:

1. Upload the .docx template to Workato FileStorage
2. Call `extract_merge_fields` on the uploaded template
3. Pass the tag list + tables + col_tables output to the AI agent with this prompt:
   > "Generate a data contract for this template. For each field, document:
   > the field name, prefix type, data source, expected format, and
   > whether it is AI-authored or a data lookup. Group by section."
4. Save the generated data contract as a companion `.md` file
   alongside the template in FileStorage
5. Reference the data contract in the Workato recipe system prompt

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
- `{{IMG:...}}`, `{{HTML:...}}`, `{{TABLE_START:...}}`, `{{TABLE_END:...}}`, 
  and `{{PAGE_BREAK}}` must each be on their own paragraph
- `{{IMG:...}}` width hint: append `__Xcm` to the tag name (e.g. `{{IMG:graph__15cm}}`). The payload key is always the name without the hint (e.g. `IMG:graph`)
- Plain text and `{{AI:...}}` fields can appear inline anywhere, including mid-sentence
- `TABLE_ROWS:` column names in the payload JSON must exactly match 
  `{{column_name}}` tags in the Word template row — case-sensitive, no spaces
- `TABLE_COLS:` key names in each payload object must exactly match the 
  `{{placeholder}}` names in the template's last column — case-sensitive, no spaces
- For `TABLE_COLS:` tables, place `{{TABLE_COLS:name}}` in the header cell of 
  the template column. No `{{TABLE_START:name}}` / `{{TABLE_END:name}}` markers needed
- Conditional blocks have two modes:
  - **Block mode:** `{{#name}}` and `{{/name}}` each on their own paragraph
  - **Inline mode:** `{{#name}}` and `{{/name}}` in the same paragraph
- Nested conditional blocks are supported as long as names are unique
- `{{PAGE_BREAK}}` requires no payload entry and will not appear in the tag list
- Place `{{PAGE_BREAK}}` immediately before `{{TABLE_START:name}}` to prevent 
  page-break gaps around tables

---

## Template Examples

### Example 1 — Client Letter

```
{{client_name}}
{{client_address}}
{{letter_date}}
Dear {{client_salutation}},
{{HTML:letter_body}}
Yours sincerely,
{{IMG:adviser_signature}}
{{adviser_name}}
{{adviser_title}}
```

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

```
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
```

```json
{
  "fields": [
    { "key": "client_name",               "value": "John Smith" },
    { "key": "soa_date",                  "value": "1 May 2026" },
    { "key": "AI:executive_summary",      "value": "This SOA has been prepared for John Smith following our review on 28 April 2026." },
    { "key": "#show_partner_section",     "value": "true" },
    { "key": "partner_name",              "value": "Jane Smith" },
    { "key": "partner_dob",               "value": "15/06/1972" },
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

```
The recommended {{mda_portfolio_name}} Portfolio is appropriate for you
because {{AI:mda_portfolio_rationale}}.
```

```json
{ "key": "mda_portfolio_name",         "value": "Akambo Balanced" },
{ "key": "AI:mda_portfolio_rationale", "value": "it aligns with your Growth risk profile and 15-year investment horizon" }
```

---

### Example 4 — Inline Conditionals

```
This arrangement will be between {{client1_name}} & {{client2_name}}{{#show_entity_party}}, and {{entity_name}}{{/show_entity_party}}, and Intergen Advisory Partners Pty Ltd.
```

```json
{ "key": "#show_entity_party", "value": "false" }
```

Output: `This arrangement will be between John Smith & Jane Smith, and Intergen Advisory Partners Pty Ltd.`

---

### Example 5 — Descriptive Conditional Names

Block names can be long and descriptive — they act as instructions to the agent:

```
{{#has_asset_allocation_variances_exceeding_10_percent}}
Asset Allocation Variance Explanation
{{HTML:asset_allocation_variance_explanation}}
{{/has_asset_allocation_variances_exceeding_10_percent}}

{{#client_has_been_declared_bankrupt_in_last_7_years}}
{{AI:bankruptcy_disclosure_narrative}}
{{/client_has_been_declared_bankrupt_in_last_7_years}}
```

---

### Example 6 — HTML field with inline table

```json
{ "key": "HTML:portfolio_analysis", "value": "<p>Based on your <strong>Balanced</strong> risk profile:</p><table><tr><th>Asset Class</th><th>Recommended</th></tr><tr><td>Australian Shares</td><td>35%</td></tr><tr><td>International Shares</td><td>25%</td></tr></table><p>This allocation targets <em>moderate growth</em> over <strong>15 years</strong>.</p>" }
```

---

### Example 7 — Column-repeating table (product comparison)

The template has fixed row labels in column 1, `{{TABLE_COLS:name}}` in the 
header cell of column 2, and `{{placeholders}}` in the data cells of column 2. 
Subheader rows have an empty cell in column 2 — the engine clones it with its 
styling intact and no text.

```
| Fee                  | {{TABLE_COLS:current_funds}} |
|----------------------|------------------------------|
| Product              | {{product}}                  |
| Balance              | {{balance}}                  |
| Ongoing fees         |                              |
|   Investment fee     | {{investment_fee}}           |
|   Sliding admin fee  | {{sliding_admin_fee}}        |
|   Admin fee (flat)   | {{admin_fee_flat}}           |
|   ORR levy           | {{orr_levy}}                 |
| Total product costs  | {{total_product_costs}}      |
| Total combined costs | {{total_combined_costs}}     |
```

Payload — one object per fund, key names matching the template placeholders:

```json
{
  "key": "TABLE_COLS:current_funds",
  "value": "[{\"product\": \"AMP MySuper\", \"balance\": \"$145,000\", \"investment_fee\": \"0.68%\", \"sliding_admin_fee\": \"$185 p.a.\", \"admin_fee_flat\": \"$52 p.a.\", \"orr_levy\": \"$14.50\", \"total_product_costs\": \"$1,233\", \"total_combined_costs\": \"$1,233\"}, {\"product\": \"REST Core Strategy\", \"balance\": \"$62,000\", \"investment_fee\": \"0.55%\", \"sliding_admin_fee\": \"$78 p.a.\", \"admin_fee_flat\": \"$0\", \"orr_levy\": \"$6.20\", \"total_product_costs\": \"$425\", \"total_combined_costs\": \"$425\"}, {\"product\": \"Hostplus Balanced\", \"balance\": \"$38,500\", \"investment_fee\": \"0.62%\", \"sliding_admin_fee\": \"$56 p.a.\", \"admin_fee_flat\": \"$0\", \"orr_levy\": \"$3.85\", \"total_product_costs\": \"$295\", \"total_combined_costs\": \"$295\"}]"
}
```

`extract_merge_fields` output for this table:

```json
{
  "col_tables": [
    {
      "name": "current_funds",
      "type": "TABLE_COLS",
      "payload_key": "TABLE_COLS:current_funds",
      "row_keys": "admin_fee_flat, balance, investment_fee, orr_levy, product, sliding_admin_fee, total_combined_costs, total_product_costs"
    }
  ]
}
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
- **HTML tables** — pass a `<table>` tag inside an `HTML:` field value to inject 
  a native Word table. Header cells (`<th>`) receive bold text and a light grey 
  background automatically.
- **TABLE_COLS subheader rows** — rows with an empty template cell are cloned 
  with their Word styling (shading, borders) intact but no text injected. Subheader 
  rows, spacer rows, and section labels render correctly across all generated 
  columns without any special syntax.
- **TABLE_COLS column count** — the number of columns generated equals the number 
  of objects in the payload array. There is no hard limit.
- **Column hiding** — conditional blocks operate at the paragraph level. Hiding 
  an entire table column is not supported. Pass empty values instead.
- **Unresolved tags** — fields present in the template but missing from the 
  payload appear in `unresolved_tags` rather than being silently removed.
- **Blank page removal** — when a `false` conditional block is removed, any 
  page-break paragraph immediately before it is automatically collapsed so blank 
  pages are never produced.
