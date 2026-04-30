# DOCX Merge Engine

A Python-based document generation engine that populates Word (`.docx`) templates with dynamic data via a key-value API. Two functions work together: one retrieves the fields required by a template, the other merges data into it and returns the populated document.

Designed to run as a Workato Python action. Compatible with AI agents — the tag list returned by the retriever is self-documenting, telling the agent exactly what type of value each field expects.

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
  "count": 8,
  "tags": "#show_partner_section, AI:risk_profile_narrative, HTML:letter_body, IMG:adviser_signature, TABLE:portfolio_table, adviser_name, client_name, letter_date",
  "files_scanned": "word/document.xml"
}
```

**Files scanned**

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
| `unresolved_tags` | `array` | Fields present in the template but not supplied in the payload |
| `unresolved_count` | `integer` | Count of unresolved fields |

---

## Field Types

The engine supports seven field types, distinguished by prefix.

### Plain text — no prefix

Standard key-value replacement. Value is XML-escaped and inserted as plain text. Can appear inline within sentences, inside table cells, or anywhere in the document.

**Template**
```
Dear {{client_salutation}},

Your adviser is {{adviser_name}}.
```

**Payload**
```json
{ "key": "client_salutation", "value": "John" },
{ "key": "adviser_name", "value": "Michael Chen" }
```

---

### Conditional blocks — `#` prefix

Show or hide entire sections based on a `true`/`false` value. Opening and closing markers must each sit **alone on their own line**.

**Template**
```
{{#show_partner_section}}
Partner name:   {{partner_name}}
Date of birth:  {{partner_dob}}
{{/show_partner_section}}
```

**Payload**
```json
{ "key": "show_partner_section", "value": "true" },
{ "key": "partner_name", "value": "Jane Smith" },
{ "key": "partner_dob", "value": "15/06/1972" }
```

When `false` or omitted, the entire block including all content between the markers is removed. When `true`, the markers are removed and the content is retained.

Conditional block names can be long and descriptive — the name itself acts as an instruction to the agent:

```
{{#client_has_existing_insurance_policies}}
...
{{/client_has_existing_insurance_policies}}

{{#partner_is_retired_or_approaching_retirement}}
...
{{/partner_is_retired_or_approaching_retirement}}
```

> **Rule:** `{{#block_name}}` and `{{/block_name}}` must each be on their own paragraph. They cannot share a line with other text.

---

### Image — `IMG:` prefix

Fetches an image from a URL and embeds it inline at the placeholder location. Supports PNG, JPEG, GIF, BMP, and WebP. PNG is recommended for signatures as it supports transparency.

**Template**
```
{{IMG:adviser_signature}}
```

**Payload**
```json
{ "key": "IMG:adviser_signature", "value": "https://yourbucket.s3.amazonaws.com/sigs/adviser.png" }
```

The image is fetched at merge time and sized using its actual pixel dimensions at 96dpi. The URL must be publicly accessible or a presigned link. If unreachable, the field is replaced with `[Image unavailable: <reason>]` and processing continues.

> **Rule:** `{{IMG:field_name}}` must sit alone on its own line in Word.

---

### HTML table — `TABLE:` prefix

Converts an HTML table string to a native Word table and injects it at the placeholder location. Header cells (`<th>`) automatically receive bold text and a light grey background.

**Template**
```
{{TABLE:portfolio_table}}
```

**Payload**
```json
{ "key": "TABLE:portfolio_table", "value": "<table><tr><th>Fund</th><th>Value</th><th>Allocation</th></tr><tr><td>Australian Shares</td><td>$120,000</td><td>40%</td></tr><tr><td>International Shares</td><td>$90,000</td><td>30%</td></tr></table>" }
```

> **Rule:** `{{TABLE:field_name}}` must sit alone on its own line. Wrap in a conditional block to make it optional.

---

### Rich HTML content — `HTML:` prefix

Converts a rich HTML string to native Word XML and injects it at the placeholder location. Supports:

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
| `<table>` | Native Word table (same as `TABLE:`) |

Heading styles (`Heading1`, `Heading2` etc.) resolve against the styles defined in the template — font, colour, and spacing are controlled by the template, not the engine.

A single `HTML:` field can contain any mix of paragraphs, headings, lists, bold, italic, and tables in any order.

**Template**
```
{{HTML:letter_body}}
```

**Payload**
```json
{ "key": "HTML:letter_body", "value": "<h2>Our Recommendations</h2><p>Based on our review, we recommend the following:</p><ol><li><strong>Consolidate superannuation</strong> — reduce fees by merging three accounts into one.</li><li><strong>Increase salary sacrifice</strong> — contribute <strong>$27,500 p.a.</strong> to reduce taxable income.</li></ol><p>We will prepare a full Statement of Advice outlining these strategies.</p>" }
```

> **Rule:** `{{HTML:field_name}}` must sit alone on its own line.

---

### AI narrative — `AI:` prefix

Semantically identical to a plain text field — the value is inserted as-is. The `AI:` prefix is a signal to the agent that this field requires authored narrative content rather than a data lookup. For formatted output use `HTML:` instead.

**Template**
```
{{AI:executive_summary}}
{{AI:risk_profile_rationale}}
{{AI:advice_limitations}}
```

**Payload**
```json
{ "key": "AI:executive_summary", "value": "Based on our review of your financial position, we recommend a balanced growth strategy..." }
```

The agent reads `AI:executive_summary` in the tag list and knows to compose appropriate content based on available client context rather than retrieving a stored value.

---

### Page break — `{{PAGE_BREAK}}`

Inserts a Word page break at the placeholder location. No payload entry required — presence in the template is sufficient. Does not appear in the tag list returned by the retriever.

**Template**
```
{{PAGE_BREAK}}
```

> **Rule:** `{{PAGE_BREAK}}` must sit alone on its own line.

---

## Field Type Summary

| Field | Payload value | Own line required |
|---|---|---|
| `{{field_name}}` | Any string | No |
| `{{#block_name}}` / `{{/block_name}}` | `true` / `false` | Yes |
| `{{IMG:field_name}}` | Image URL | Yes |
| `{{TABLE:field_name}}` | HTML table string | Yes |
| `{{HTML:field_name}}` | Rich HTML string | Yes |
| `{{AI:field_name}}` | Any string (agent-authored) | No |
| `{{PAGE_BREAK}}` | None required | Yes |

---

## Template Examples

### Example 1 — Client Letter

A branded letter where the header and footer contain the logo, practice details, and disclaimer. The agent authors the full body as a single `HTML:` field.

**Template**
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

**Payload**
```json
{
  "fields": [
    { "key": "client_name",           "value": "Mr John Smith" },
    { "key": "client_address",        "value": "12 Main Street, Sydney NSW 2000" },
    { "key": "letter_date",           "value": "30 April 2026" },
    { "key": "client_salutation",     "value": "John" },
    { "key": "HTML:letter_body",      "value": "<p>Thank you for meeting with us on 28 April 2026.</p><h2>Our Recommendations</h2><p>Following our review we recommend the following strategies:</p><ol><li><strong>Consolidate superannuation</strong> — merge three accounts into one low-cost fund.</li><li><strong>Increase salary sacrifice</strong> — contribute $27,500 p.a. to reduce taxable income.</li><li><strong>Review insurance</strong> — increase life cover from $500,000 to $1,200,000.</li></ol><p>We will prepare a full Statement of Advice shortly. Please do not hesitate to contact our office if you have any questions.</p>" },
    { "key": "IMG:adviser_signature", "value": "https://yourbucket.s3.amazonaws.com/sigs/adviser.png" },
    { "key": "adviser_name",          "value": "Michael Chen" },
    { "key": "adviser_title",         "value": "Financial Adviser" }
  ]
}
```

---

### Example 2 — Statement of Advice (SOA)

A multi-section advice document with conditional partner section, portfolio table, rich recommendations, and page breaks.

**Template**
```
{{client_name}}
{{soa_date}}

{{AI:executive_summary}}

{{PAGE_BREAK}}

Personal Details

{{#show_partner_section}}
Partner name:   {{partner_name}}
Date of birth:  {{partner_dob}}
Risk profile:   {{partner_risk_profile}}
{{/show_partner_section}}

{{PAGE_BREAK}}

Recommended Portfolio

{{#show_portfolio_table}}
{{TABLE:portfolio_table}}
{{/show_portfolio_table}}

{{PAGE_BREAK}}

Our Recommendations

{{HTML:recommendations_body}}

{{PAGE_BREAK}}

{{#show_smsf_section}}
Self-Managed Super Fund

{{AI:smsf_strategy_narrative}}
{{/show_smsf_section}}

Authority to Proceed

{{IMG:client_signature}}
{{client_name}}
```

**Payload**
```json
{
  "fields": [
    { "key": "client_name",                "value": "John Smith" },
    { "key": "soa_date",                   "value": "30 April 2026" },
    { "key": "AI:executive_summary",       "value": "This Statement of Advice has been prepared for John Smith following our review meeting on 28 April 2026. It sets out our recommendations across superannuation, insurance, and investment." },
    { "key": "show_partner_section",       "value": "true" },
    { "key": "partner_name",               "value": "Jane Smith" },
    { "key": "partner_dob",                "value": "15/06/1972" },
    { "key": "partner_risk_profile",       "value": "Balanced" },
    { "key": "show_portfolio_table",       "value": "true" },
    { "key": "TABLE:portfolio_table",      "value": "<table><tr><th>Asset Class</th><th>Current</th><th>Recommended</th></tr><tr><td>Australian Shares</td><td>20%</td><td>35%</td></tr><tr><td>International Shares</td><td>10%</td><td>25%</td></tr><tr><td>Fixed Income</td><td>50%</td><td>25%</td></tr><tr><td>Cash</td><td>20%</td><td>15%</td></tr></table>" },
    { "key": "HTML:recommendations_body",  "value": "<h2>Superannuation</h2><p>We recommend consolidating your three existing accounts into a single low-cost fund to reduce fees and simplify management.</p><h2>Insurance</h2><p>Your current life cover of <strong>$500,000</strong> is insufficient. We recommend increasing this to <strong>$1,200,000</strong> given your current liabilities.</p><h2>Debt Management</h2><p>Redirecting surplus cashflow of <strong>$1,500 per month</strong> to your mortgage will reduce your loan term by an estimated <em>six years</em>.</p>" },
    { "key": "show_smsf_section",          "value": "false" },
    { "key": "IMG:client_signature",       "value": "https://yourbucket.s3.amazonaws.com/sigs/john-smith.png" }
  ]
}
```

---

### Example 3 — Descriptive Conditional Names

Block names can be long and descriptive — they act as instructions to the agent reading the tag list without requiring any additional documentation.

**Template**
```
{{#client_has_existing_superannuation_balance_over_500k}}
Your current superannuation balance places you in a strong position for retirement.
{{/client_has_existing_superannuation_balance_over_500k}}

{{#client_has_been_declared_bankrupt_in_last_7_years}}
{{AI:bankruptcy_disclosure_narrative}}
{{/client_has_been_declared_bankrupt_in_last_7_years}}

{{#partner_is_retired_or_within_5_years_of_retirement}}
Transition to retirement strategies may be appropriate for {{partner_name}}.
{{/partner_is_retired_or_within_5_years_of_retirement}}
```

---

### Example 4 — HTML Field with Inline Table

A single `HTML:` field containing paragraphs, a table, and formatted text — no separate `TABLE:` field needed.

**Payload**
```json
{
  "fields": [
    { "key": "HTML:portfolio_analysis", "value": "<p>Based on your <strong>Balanced</strong> risk profile, we recommend the following asset allocation:</p><table><tr><th>Asset Class</th><th>Recommended</th><th>Rationale</th></tr><tr><td>Australian Shares</td><td>35%</td><td>Long-term growth</td></tr><tr><td>International Shares</td><td>25%</td><td>Diversification</td></tr><tr><td>Fixed Income</td><td>25%</td><td>Stability</td></tr><tr><td>Cash</td><td>15%</td><td>Liquidity</td></tr></table><p>This allocation is designed to achieve <em>moderate growth</em> while managing downside risk appropriate to your investment horizon of <strong>15 years</strong>.</p>" }
  ]
}
```

---

## Template Authoring Guide

### Word styles

Heading styles in `HTML:` fields resolve against the styles defined in your template. Define the following styles in your base template to ensure consistent rendering:

| Style name | Used for |
|---|---|
| `Heading 1` — `Heading 4` | `<h1>` — `<h4>` in HTML fields |
| `List Bullet` | `<ul>` / `<li>` |
| `List Number` | `<ol>` / `<li>` |
| `Normal` | `<p>` and default paragraphs |

The engine injects Word style names — font, colour, and spacing are always controlled by the template, never the payload.

### Workflow

1. Build the template in Word with all placeholders in place
2. Call `extract_merge_fields` to get the full tag list
3. Build the payload — use the prefix to determine what each field expects:
   - `#field` → `true` / `false`
   - `IMG:field` → image URL
   - `TABLE:field` → HTML table string
   - `HTML:field` → rich HTML string
   - `AI:field` → agent-authored narrative string
   - plain field → any string
4. Call `merge_document` with the template and payload
5. Decode `merged_docx_base64` to retrieve the final document
6. Check `unresolved_tags` — any entries mean the payload was missing a field the template expected

### Rules summary

- Field names may only contain letters, numbers, and underscores
- `{{#...}}`, `{{/...}}`, `{{IMG:...}}`, `{{TABLE:...}}`, `{{HTML:...}}`, and `{{PAGE_BREAK}}` must each be on their own line
- Plain text and `{{AI:...}}` fields can appear inline anywhere
- Conditional blocks can wrap any content including tables, images, and HTML fields
- Nested conditional blocks are supported as long as names are unique
- `{{PAGE_BREAK}}` requires no payload entry and will not appear in the retrieved tag list

---

## Dependencies

| Library | Workato version | Purpose |
|---|---|---|
| `Pillow` | 11.3.0 | Reading image dimensions for embedding |
| `lxml` | 6.0.2 | HTML parsing and Word XML generation |
| `requests` | 2.32.5 | Fetching images from URLs |
| `zipfile` | stdlib | Reading and writing the docx zip format |
| `re` | stdlib | XML processing and field extraction |
| `base64` | stdlib | Encoding/decoding file contents |

All libraries are available natively in the Workato Python connector. No additional installation required.

---

## Notes

- **Split run repair** — Word sometimes splits a placeholder across multiple XML runs when typed directly in the document. The `merge_split_tags` step detects and repairs these automatically before substitution.
- **XML safety** — all plain text values are XML-escaped before injection. `&`, `<`, and `>` in values will not corrupt the document.
- **Image fallback** — if an image URL is unreachable the field is replaced with `[Image unavailable: <reason>]` and the merge continues rather than failing.
- **Table fallback** — if HTML table conversion fails the field is replaced with `[Table error: <reason>]` and the merge continues.
- **Column hiding** — conditional blocks operate at the paragraph level. Hiding an entire table column is not currently supported. Pass empty values or pre-render the table without the column as a `TABLE:` field.
- **Unresolved tags** — fields present in the template but missing from the payload appear in `unresolved_tags` rather than being silently removed, so gaps are always visible in the response.
