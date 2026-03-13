---
name: gads-editor-csv-creator
description: >
  Generate a Google Ads Editor CSV import file from analysis and RSA Markdown files.
  ALWAYS use this skill when the user mentions Google Ads Editor, CSV export, import file,
  or anything about turning ad copy / RSA files / keyword clusters into a file that Google Ads
  Editor can consume. This is the final step in the gads toolchain – after
  gads-landing-page-analyzer and gads-responsive-search-ads-creator have produced their
  Markdown output, this skill turns it into a ready-to-import CSV with campaigns, ad groups,
  keywords, RSA ads, and negative keywords. Trigger on any of these patterns: "CSV",
  "Google Ads Editor", "import file", "importfil", "Editor CSV", "export to Editor",
  "skapa CSV", "create the CSV", "gör CSV", "make the import file", "GAds Editor",
  "bulk import", "editor import", or simply "now create the file" / "exportera" /
  "generate the file" after RSA ads have been created. Also trigger when the user says
  "next step" or "now what" after running the RSA creator, since this IS the next step.
  When in doubt, trigger – this skill handles all Google Ads Editor CSV generation.
---

# Google Ads Editor CSV Creator

You generate a single CSV file that can be imported into Google Ads Editor to create complete Search campaigns: campaign settings, location targeting, ad groups, keywords, RSA ads, and negative keywords – all from the Markdown files produced by the earlier skills in the chain.

## Context: Where This Fits

This skill is the third and final link in a toolchain:

1. **gads-landing-page-analyzer** – Analyzes a landing page and produces a structured Markdown file with target segments, offers, keyword clusters, and negative keywords.
2. **gads-responsive-search-ads-creator** – Reads the analysis and produces one Markdown file per keyword cluster with complete RSA copy (headlines, descriptions, keywords, URLs, etc.).
3. **gads-editor-csv-creator** (this skill) – Reads the analysis file and all RSA files and produces one CSV file ready for Google Ads Editor import.

## Input Format

The skill reads a folder containing:

1. **One analysis file** (Markdown) – produced by gads-landing-page-analyzer. Contains negative keywords (in the "Negativa sökord" or "Negative keywords" section), landing page URL, geographic scope, and target segments.

2. **One or more RSA files** (Markdown) – produced by gads-responsive-search-ads-creator. Each file describes an ad group with this structure:

```
Kampanj: <name>                          (or "Campaign:")
Annonsgrupp: <name>                      (or "Ad group:")
Sökord: "kw1", "kw2", [kw3], kw4        (or "Keywords:")
Platsinriktning: <geo targeting>         (or "Location targeting:")
Slutlig webbadress: <URL>               (or "Final URL:")
Visningsadress – nivå 1: <max 15 chars> (or "Display path – level 1:")
Visningsadress – nivå 2: <max 15 chars> (or "Display path – level 2:")
Rubrik N (P): <text>                    (or "Headline N (P):")
Beskrivning N (P): <text>              (or "Description N (P):")
```

A single RSA file may contain **multiple ads** for the same ad group (max 3), separated by `---`. Each ad after the first inherits the ad group's campaign, ad group name, keywords, and targeting from the header – only headlines, descriptions, display paths, and final URL vary.

### Parsing Rules

Every line containing `: ` (colon followed by space) is a key-value pair. The value after `: ` is copied verbatim (stripped of leading/trailing whitespace).

**Keyword match types** are determined by the surrounding characters:
- `"keyword"` → Phrase match
- `[keyword]` → Exact match
- `keyword` (no surrounding quotes or brackets) → Broad match

**Headline positions** map from the parenthetical to CSV values:
- `(valfri position)` or `(any position)` → empty (unpinned)
- `(position 1)` → `1`
- `(position 2)` → `2`
- `(position 3)` → `3`

**Description positions** follow the same logic (only positions 1 and 2 are valid for descriptions).

## Workflow

### Phase 1: Gather Prerequisites (interactive)

Ask all necessary questions in **a single message**. Adapt based on what is already known from the conversation.

**1. Input folder**

If the user hasn't pointed to a folder containing analysis and RSA files, ask where they are. If you can see them in the workspace already, confirm the path.

**2. Daily budget**

Ask: *"What daily budget do you want for the campaign(s)? Enter an amount (e.g. 200). Leave blank if you prefer to set the budget manually after importing into Google Ads Editor."*

**3. Tracking settings**

Present three options:
- **Option 1 (default):** Leave empty – use the account's default tracking settings.
- **Option 2:** Use a suggested UTM template that works with GA4, Matomo, and other tools:
  - Tracking template: `{lpurl}`
  - Final URL suffix: `utm_source=google&utm_medium=cpc&utm_campaign={campaignid}&utm_term={keyword}&utm_content={creative}&mtm_group={adgroupid}&mtm_cid={gclid}`
- **Option 3:** Enter your own tracking template and/or suffix.

**4. Summary and confirmation**

Before generating, show a summary of all settings that will be used:
- Bid strategy: Maximize clicks
- Networks: Google Search only
- Status: Paused (campaign, ad groups, keywords, ads)
- Negative keywords: Campaign level, phrase match
- Language targeting: [detected from analysis/keywords]
- Location targeting: [from RSA files]
- Count: N campaigns, N ad groups, N keywords, N ads found

Ask the user to confirm or suggest changes. If changes are requested, show an updated summary and ask again.

### Phase 2: Generate CSV (autonomous)

Once the user confirms, work without interruption:

1. **Parse all RSA files.** Extract campaign name, ad group names, keywords with match types, headlines with positions, descriptions with positions, URLs, and display paths. Use the bundled script for reliable parsing:

```bash
python <skill-path>/scripts/generate_csv.py <input-folder> [options]
```

The script handles all parsing, validation, and CSV generation. See the Script Reference section below for usage details.

2. **Parse the analysis file** to extract negative keywords from the "Negativa sökord" or "Negative keywords" section.

3. **Validate character limits:**
   - Headlines: max 30 characters
   - Descriptions: max 90 characters
   - Display paths: max 15 characters each

   If violations are found, **warn the user** in the chat message but still generate the CSV. List all violations so the user can fix them.

4. **Detect language** from the keywords and analysis. Map to Google Ads language code (e.g. Swedish → `sv`, English → `en`).

5. **Map location targeting** to Google Ads Location IDs. Common mappings:
   - "Nationell" / "Sverige" / "Sweden" → `2752`
   - "Stockholm" → `1012728`
   - "Göteborg" / "Gothenburg" → `1011919`
   - "Malmö" → `1012046`
   - "Finland" → `2246`
   - "Norge" / "Norway" → `2578`
   - "Danmark" / "Denmark" → `2208`

   If the location cannot be mapped, use the `Location` column with the place name instead of `Location ID`.

6. **Generate the CSV file** with all rows in hierarchical order.

7. **Save the CSV file** with name `<CampaignName>_<YYYY-MM-DD>.csv` (campaign name slugified, today's date).

8. **Present the result** with a summary including: filename, counts (campaigns, ad groups, keywords by match type, ads, negative keywords), any warnings, and import instructions.

## CSV Output Format

### General Rules

- Standard Google Ads Editor column headers (English) on the first row.
- Comma-separated values.
- UTF-8 encoding with BOM (`\xEF\xBB\xBF`) for compatibility with Excel and Google Ads Editor.
- Fields containing commas, quotes, or newlines are quoted; internal quotes are doubled.
- Leave fields empty (not `[]`) for values that should not be changed.

### Row Types and Order (hierarchical)

The CSV contains these row types in this order:

1. **Campaign row** – one per campaign.
2. **Location targeting rows** – one per location, directly after the campaign row.
3. **Per ad group (repeat for each):**
   a. **Ad group row**
   b. **Keyword rows** – one per keyword
   c. **Ad row(s)** – one per RSA ad (typically 1, max 3)
4. **Campaign-level negative keywords** – all negatives from the analysis, at the end of the file.

### Columns by Row Type

#### Campaign Row

| Column | Value |
|--------|-------|
| Campaign | Campaign name from RSA files |
| Campaign type | `Search` |
| Campaign daily budget | User-specified budget (or empty) |
| Language | Language code detected from analysis (e.g. `sv`) |
| Networks | `Google Search` |
| Bid strategy type | `Maximize clicks` |
| Campaign status | `Paused` |
| Tracking template | Per user choice |
| Final URL suffix | Per user choice |

#### Location Targeting Row

| Column | Value |
|--------|-------|
| Campaign | Campaign name |
| Location ID | Google Ads Location ID (e.g. `2752` for Sweden) |

If Location ID cannot be determined, use the `Location` column with the place name instead.

#### Ad Group Row

| Column | Value |
|--------|-------|
| Campaign | Campaign name |
| Ad Group | Ad group name from RSA file |
| Ad Group status | `Paused` |

#### Keyword Row (one per keyword)

| Column | Value |
|--------|-------|
| Campaign | Campaign name |
| Ad Group | Ad group name |
| Keyword | The keyword text (without surrounding quotes or brackets) |
| Criterion Type | `Broad`, `Phrase`, or `Exact` |
| Status | `Paused` |

#### Ad Row (RSA)

| Column | Value |
|--------|-------|
| Campaign | Campaign name |
| Ad Group | Ad group name |
| Ad Name | `{AdGroup}_RSA_1` (or `_RSA_2`, `_RSA_3` for additional ads) |
| Ad type | `Responsive search ad` |
| Status | `Paused` |
| Final URL | Final URL from RSA file |
| Path 1 | Display path level 1 |
| Path 2 | Display path level 2 |
| Headline 1 | Headline 1 text |
| Headline 1 position | Position value (empty/1/2/3) |
| ... | ... (up to Headline 15 + position) |
| Description 1 | Description 1 text |
| Description 1 position | Position value (empty/1/2) |
| ... | ... (up to Description 4 + position) |

#### Negative Keyword Row (campaign level)

| Column | Value |
|--------|-------|
| Campaign | Campaign name |
| Keyword | The negative keyword in phrase match format (wrapped in `"..."`) |
| Criterion Type | `Campaign negative` |

Negative keywords default to phrase match. Each keyword is wrapped in quotation marks in the Keyword column.

## Script Reference

The bundled Python script `scripts/generate_csv.py` handles all parsing, validation, and CSV generation. It is the backbone of this skill – use it rather than writing CSV generation logic from scratch.

**Usage:**

```bash
python <skill-path>/scripts/generate_csv.py \
  --input-dir <folder-with-md-files> \
  --output-dir <where-to-save-csv> \
  [--budget <daily-budget>] \
  [--tracking-template <template>] \
  [--final-url-suffix <suffix>] \
  [--language <lang-code>] \
  [--location-id <id>] \
  [--location-name <name>]
```

The script:
- Auto-detects which file is the analysis (looks for "Negativa sökord" / "Negative keywords" section) and which are RSA files (looks for "Kampanj:" / "Campaign:" on the first line).
- Parses all key-value pairs from RSA files, including multi-ad files separated by `---`.
- Extracts negative keywords from the analysis file.
- Validates character limits and prints warnings to stderr.
- Generates the CSV with proper encoding (UTF-8 BOM), quoting, and column order.
- Outputs the filename to stdout.

If `--language` is not specified, the script detects it from the keyword content. If `--location-id` is not specified, it reads the "Platsinriktning" / "Location targeting" field from RSA files and attempts to map it.

## Design Principles

This skill is opinionated – it follows best practices as defaults and requires the user to actively choose to deviate:

- **One campaign, many ad groups** is the standard structure. Each ad group has 1–3 RSA ads and a cluster of thematically related keywords.
- **Phrase match is the default** for regular keywords (marked with `"..."` in the RSA files).
- **Exact match** for keywords marked with `[...]`.
- **Broad match** for keywords without any surrounding characters.
- **Maximize clicks** as the starting bid strategy – with a recommendation to switch to Maximize conversions or Target CPA once the account has 30+ conversions per month.
- **Google Search only** – no Search Partners, for maximum control.
- **Everything paused** at creation – for review before activation.
- **Negative keywords at campaign level** with phrase match.

## Future Extension (v2)

A future version will support **updating existing campaigns** using `#Original` columns. This requires the user to provide an exported CSV from Google Ads Editor (current state) alongside updated RSA files. The skill will compare and produce `#Original` columns for changed fields. This is not in scope for v1.

## Language

Respond in the same language the user uses. The CSV file always uses English column headers (Google Ads Editor requires this).

## Typography

Use proper Unicode characters in all output:
- En dash: – (U+2013)
- Quotation marks: "..." (U+201C/U+201D)
- Arrow: → (U+2192)
