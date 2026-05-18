# Product Requirements Document
## DOCX Template Builder Server

**Version:** 1.0  
**Date:** 18 May 2026  
**Status:** Draft

---

## 1. Overview

### 1.1 Purpose

The Template Builder Server is an AI-powered MCP server that generates Word (.docx) merge templates from a natural language description or structured spec. It is the companion to the DOCX Merge Engine — where the merge server populates templates, the builder server creates them.

### 1.2 Problem

Currently, templates must be hand-authored in Word by a paraplanner or template author who understands the merge engine's field syntax. This creates a bottleneck:

- Template authors must learn `{{TABLE_COLS:}}`, `{{REPEAT_START:}}`, conditional block syntax, image hint conventions, and table setup rules
- Mistakes in template structure (wrong markers, missing `TABLE_START`/`TABLE_END`, columns not on their own line) cause silent merge failures
- Onboarding a new template takes hours of manual work and testing
- There is no way to programmatically generate a template from a data model

### 1.3 Solution

An AI agent that reads the Template Authoring Guide, understands the full field type system, and generates a correctly structured `.docx` template from a description. The agent handles all the structural complexity — the human describes what they want and receives a ready-to-use template.

---

## 2. Goals

- Reduce template creation time from hours to minutes
- Eliminate structural authoring errors (missing markers, wrong column setup, etc.)
- Make template creation accessible to non-technical users
- Produce templates that are immediately usable with the merge server with no manual fixup
- Generate a companion data contract alongside every template

### 2.1 Out of scope (v1)

- Editing or updating existing templates (create-only)
- Custom styling beyond a defined set of house style options
- Multi-licensee branding in a single template
- Template versioning or diff

---

## 3. Users

| User | Description |
|---|---|
| Paraplanner | Describes what the SOA/ROA/letter needs, receives a template ready to populate |
| Practice manager | Creates standardised templates for their adviser group |
| Developer / recipe builder | Programmatically generates templates as part of a document workflow |
| AI agent (Workato recipe) | Primary consumer of the `create_docx_template` tool |

---

## 4. Tools

### 4.1 `get_template_authoring_guide`

Returns the full `TEMPLATE_AUTHORING_GUIDE.md` as a string.

**Input:** None  
**Output:** Markdown string

The agent must call this before calling `create_docx_template`. It provides the authoritative reference for field types, naming conventions, table layout rules, image sizing, and Word style requirements.

**Tool description:**
> Returns the DOCX template authoring guide. Call this before create_docx_template to understand all supported field types, table layout conventions, conditional block syntax, image hints, and Word style requirements.

---

### 4.2 `create_docx_template`

Builds a `.docx` template from a structured spec, saves it to FileStorage, and returns the file path plus `extract_merge_fields` output as a companion data contract.

**Input**

```json
{
  "template_name": "string — filename without extension, e.g. FWP_SOA_Template",
  "output_path": "string — FileStorage path, e.g. /Word Templates/",
  "house_style": "string — one of: intergen | fwp | hph | annex | default",
  "header": {
    "left_text": "string — static text, e.g. Statement of Advice",
    "right_fields": ["client_name", "current_date"]
  },
  "footer": {
    "left_fields": ["licensee_name"],
    "right_fields": ["current_date"]
  },
  "sections": [
    {
      "title": "string — section heading text",
      "heading_level": "integer — 1 or 2",
      "page_break_before": "boolean",
      "type": "string — one of: standard | repeat | conditional",
      "conditional_flag": "string — only for type=conditional, e.g. #show_partner_section",
      "repeat_name": "string — only for type=repeat, e.g. recommendations",
      "fields": [
        {
          "name": "string — field name, e.g. client_name or AI:rationale",
          "label": "string — optional label for two-column layout, e.g. Client name",
          "layout": "string — one of: inline | labelled | standalone"
        }
      ],
      "tables": [
        {
          "type": "string — TABLE_ROWS or TABLE_COLS",
          "name": "string — table name, e.g. ongoing_fees",
          "page_break_before": "boolean",
          "columns": ["string — for TABLE_ROWS: column key names"],
          "column_headers": ["string — for TABLE_ROWS: display header text"],
          "row_labels": [
            {
              "label": "string — display text in col 1",
              "key": "string or null — placeholder key for col 2, null for subheader rows",
              "style": "string — one of: normal | subheader | bold | indent"
            }
          ]
        }
      ],
      "images": [
        {
          "name": "string — e.g. adviser_signature",
          "width_cm": "number",
          "label": "string — optional label above image, e.g. Signature:"
        }
      ]
    }
  ]
}
```

**Output**

```json
{
  "success": true,
  "template_file_path": "/Word Templates/FWP_SOA_Template.docx",
  "template_download_url": "https://...",
  "data_contract": {
    "count": 42,
    "tags": "...",
    "tables": [...],
    "col_tables": [...],
    "repeating_sections": [...]
  }
}
```

**Tool description:**
> Generates a Word (.docx) merge template from a structured spec and saves it to FileStorage. Call get_template_authoring_guide first. Returns the file path and a data contract (extract_merge_fields output) so the template is immediately usable with the merge server. Supports all field types: plain text, AI narrative, HTML content, images with sizing hints, conditional blocks, TABLE_ROWS, TABLE_COLS, and REPEAT sections.

---

### 4.3 `get_all_templates`

Shared with the merge server. Lists all templates in FileStorage so the agent can check what already exists.

---

## 5. House styles

Each house style defines a colour palette, font, and heading style applied to the generated template. v1 ships with:

| Style key | Licensee | Primary colour | Font |
|---|---|---|---|
| `intergen` | Intergen Advisory Partners | `#1F4E45` (dark teal) | Arial |
| `fwp` | First Wealth Partners | `#1F3864` (navy) | Arial |
| `hph` | HPH Advisory | `#2E4057` (slate) | Calibri |
| `annex` | Annex Wealth | `#2C2C2C` (charcoal) | Arial |
| `default` | Generic | `#2E75B6` (blue) | Arial |

House style controls: heading colours, header/footer border colour, table header background, font family. All other formatting (paragraph spacing, margin, page size) is standardised.

---

## 6. Field layout modes

Fields in the spec can be rendered in three layouts:

| Layout | Description | Example |
|---|---|---|
| `labelled` | Two-column table row: label left, placeholder right | `Client name    {{client_name}}` |
| `inline` | Placeholder inline within a sentence | `Dear {{client_preferred_name}},` |
| `standalone` | Placeholder on its own paragraph | `{{HTML:recommendations_body}}` |

The agent selects the appropriate layout based on field type if not specified:
- `HTML:`, `IMG:`, `TABLE_ROWS:`, `TABLE_COLS:`, `REPEAT_START/END`, `PAGE_BREAK` → `standalone`
- `AI:` → `inline` by default (can be overridden to `standalone`)
- Plain text → `labelled` by default (can be overridden)

---

## 7. Python action implementation

The `create_docx_template` tool is a Workato Python action using `python-docx`.

### 7.1 What it generates

- Document styles: Heading 1–4, List Bullet, List Number, Normal — with house style colours and font
- Numbering definitions for `List Bullet` (bullets) and `List Number` (decimal) in `numbering.xml`
- Header with left static text and right field placeholders, border bottom
- Footer with left/right field placeholders, border top, right-aligned tab stop
- Each section in order: heading, content fields, tables, images
- `TABLE_ROWS` tables: header row (styled) + one template row with `{{placeholders}}`
- `TABLE_COLS` tables: label column + template column with `{{TABLE_COLS:name}}` in header, `{{placeholder}}` in data rows, empty cells for subheader rows
- `REPEAT_START` / `REPEAT_END` marker paragraphs (styled as a distinct colour so they're visible in the template)
- Conditional block markers in a distinct colour
- `{{PAGE_BREAK}}` paragraphs before tables where `page_break_before: true`
- AutoFit to Window on all tables (`<w:tblW w:w="0" w:type="auto"/>`)

### 7.2 Dependencies

| Library | Purpose |
|---|---|
| `python-docx` | Building the .docx file |
| `zipfile` | stdlib — reading/writing .docx zip |
| `base64` | stdlib — encoding for FileStorage |

### 7.3 After generation

After building the `.docx`, the action:
1. Saves the file to the specified FileStorage path
2. Calls `extract_merge_fields` internally on the generated file
3. Returns both the file path and the data contract in a single response

---

## 8. Agent behaviour

The agent orchestrating the template creation should follow this sequence:

1. Call `get_template_authoring_guide` — load field type conventions
2. Understand the user's request — what document type, what sections, what data fields are needed
3. Call `get_all_templates` — check if a similar template already exists
4. Build the spec — apply field type selection rules:
   - Variable-length lists → `TABLE_ROWS:`
   - Comparison tables (products, funds) → `TABLE_COLS:`
   - Rich variable-length content (recommendations, properties) → `REPEAT:`
   - Show/hide sections → `#conditional`
   - Formatted paragraphs → `HTML:`
   - Short inline narrative → `AI:`
5. Call `create_docx_template` with the spec
6. Return the template download URL and data contract to the user

---

## 9. MCP server description

> A template creation server for building Word (.docx) merge templates compatible with the DOCX Merge Engine. Call get_template_authoring_guide first to load field type conventions, then call create_docx_template with a structured spec to generate a correctly structured template — with all field placeholders, tables, conditional blocks, repeat sections, headers, footers, and house styling applied automatically. The template is saved directly to FileStorage and a companion data contract is returned so it is immediately usable with the merge server. Supports all merge engine field types: plain text, AI narrative, HTML content, images with sizing hints, conditional blocks, TABLE_ROWS, TABLE_COLS, and REPEAT sections.

---

## 10. Open questions

| # | Question | Impact |
|---|---|---|
| 1 | Does the spec expose logo/image in the header, or is that baked into the house style? | High — affects header implementation |
| 2 | Should the agent be able to infer the full spec from a free-text description, or does the user always provide a structured input? | High — affects UX and recipe design |
| 3 | Do we support updating existing templates (add a section, rename a field) in v1 or strictly create-only? | Medium |
| 4 | How does the agent know the `row_labels` list for a `TABLE_COLS` table without being told? E.g. for a funds comparison, does it have a library of standard row schemas? | Medium |
| 5 | Is the `house_style` selected by the recipe (hardcoded per licensee) or by the agent at runtime? | Low |
| 6 | Should generated templates be stored in a subfolder per licensee, e.g. `/Word Templates/Intergen/`? | Low |
