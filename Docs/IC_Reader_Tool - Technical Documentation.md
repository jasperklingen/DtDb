## Overview

The IC Reader Tool validates an **Information Container (IC)** file, against a configurable set of SPARQL queries, and produces a single, self-contained HTML validation report. Import can be  either a raw `.ttl` or a `.zip` of Turtle files.

At a high level it:

1. Reads query definitions and report text from an **Excel config workbook**.
2. Filters those queries down to the ones relevant to the selected **Category**.
3. Combines the supplied IC file into one RDF graph and **uploads it to Laces** as a temporary publication.
4. Runs each selected **SPARQL query** against that publication.
5. Assembles the results into a styled **HTML report**.
6. Computes two **MD5 hashes** (for traceability) and appends them to the report footer.
7. **Deletes** the temporary Laces publication.

A standalone companion page, `HashVerifier.html`, lets a reviewer independently recompute those two hashes from a report and confirm they match — without needing FME.
![[IC_Reader_Tool-System Overview.drawio.png]]

---

## Workspace

**Location:** `Workspaces/IC_Reader_Tool/IC_Reader_Tool.fmw`
**Config workbook (default):** `IC_Reader_Tool_Config_v5.xlsx`

### Published Parameters

| Parameter                         | Description                                                                                       |
| --------------------------------- | ------------------------------------------------------------------------------------------------- |
| `Category`                        | Which query set to run `vs1`, `vs2`, or `All` (`0`). `0`/blank means "run everything applicable." |
| `InformationContainer`            | Path/URL to the IC file to validate, a `.ttl` file or a `.zip` archive of `.ttl` files.           |
| `AUTH_USERNAME` / `AUTH_PASSWORD` | HTTP Basic Auth credentials used for all Laces calls (upload, SPARQL, delete).                    |
| `LACES_REPO`                      | Target Laces repository name the temporary publication is created in.                             |
| `LACES_BASE_URL`                  | Base URL of the Laces instance, e.g. `https://<host>`.                                            |
| `OUT_HTML`                        | Destination path for the generated HTML report.                                                   |

---

## FME Bookmarks

### 1. Excel Config

Reads the Excel workbook (`XLSXR` reader) and supplies the query definitions and metadata used by every later stage: `Query` (the SPARQL text), `QueryTitle`, `Explanation`, `Order`, `Category`, `Headers` (report section grouping), and `Type`.

### 2. VS selection

Filters the Excel rows so only the queries relevant to the selected `Category` run:

- `NullAttributeMapper` first normalizes a missing/null `Category` value to `0`, for backward compatibility with older config rows.
- `Tester` (`T_Catagory_Selection`) passes a row through if **any** of:
  - `@Value(Category) = $(Category)`  row matches the selected category, or
  - `@Value(Category) = 0`  row is marked "always run", or
  - `$(Category) = 0`  the user selected "All".
- A second `NullAttributeMapper` sets a missing `Headers` value to `Misc`, so ungrouped queries still get a report section.

### 3. TTL to hub.laces

Prepares and uploads the IC content to Laces as a new temporary publication:

- `Creator` + `AttributeCreator` pass the `InformationContainer` path into `PythonCaller`.
- `PythonCaller` (using `rdflib`) opens the file:
  - `.ttl` → parsed directly into an `rdflib.Graph`.
  - `.zip` → extracted to a temp directory; every `.ttl` found inside is parsed and merged into one graph.
  - The combined graph is serialized back to Turtle and written to a temp file via `TempPathnameCreator` + `AttributeFileWriter`.
- `AttributeCreator_25` builds the publication metadata JSON (`publicationUri`/`name` = run UUID, `owner: TenneT`, `publisher: FME`, `versioningMode: NONE`, empty `schemaURIs`).
- `HTTPCaller_5` `POST`s a multipart/form-data request to
  `$(LACES_BASE_URL)/api/v3/repositories/$(LACES_REPO)/publications` with two parts: the metadata JSON and the Turtle file.
- `JSONExtractor` parses the response, extracting `publication_id` (`$.id`) and `uri` (`$.uri`); everything else is dropped and these two are carried forward as `LacesPublication` / `LacesUri`.

### 4. Run SPARQL

Joins the uploaded publication details to each selected query and executes it:

- `FeatureMerger` attaches `LacesPublication` and `LacesUri` (from the upload step) onto every query row surviving the Category filter.
- `HTTPCaller_6` `POST`s the raw SPARQL text (`Content-Type: application/sparql-query`) to `$(LACES_BASE_URL)@Value(LacesUri)/sparql`, requesting `Accept: text/tab-separated-values`.
- Two `StringReplacer` transformers escape `&` → `&amp;` and `<` so the returned TSV can be safely embedded in the HTML report without breaking markup.

### 5. Validation HTML

Builds the final report:

- `HTMLReportGenerator_2` builds the report header (IC filename, selected Category).
- `HTMLReportGenerator` builds one section per query (title, explanation, TSV result table).
- `Sorter` orders all sections by the `Order` column from the config sheet.
- `HTMLLayouter` assembles the ordered sections into a single page.
- A final `StringReplacer` injects a large CSS/JS block that turns the raw TSV text into styled, sortable tables/cards and finishes the page layout.
- The **HTML Writer** writes the report to `$(OUT_HTML)`.
- The same assembled HTML is also passed to `HTTPCaller_2`, which issues a `DELETE` to `$(LACES_BASE_URL)/api/v3/publications/@Value(LacesPublication)`, removing the temporary Laces publication once the report is built.

### 6. Create hash of Filename + date as string

First traceability hash:

- Extracts the input filename, combines it with the current date (`date + filename`, no separator) as `datefilename`.
- `CRCCalculator` (MD5) hashes it → `firsthash`.

### 7. Create hash of filename + date + response bodies of all query's

Second traceability hash, sensitive to the actual query output:

- `datefilename` is renamed to `_response_body` so it can be combined with the query results.
- All rows are sorted by `Order`, line endings are normalized.
- `Aggregator` concatenates every query's response body into one string, in report order, followed by `date + filename`.
- `CRCCalculator_2` (MD5) hashes the concatenation → `secondhash`.

### 8. Make Hash footer

- `Aggregator_2` combines `firsthash` and `secondhash` onto a single feature.
- `HTMLReportGenerator_3` renders them as HTML fragments, which the injected footer CSS/JS in the Validation HTML stage displays as MD5 values at the bottom of the report.

---

## Hash Scheme (for external verification)

Both hashes are plain **MD5 of a concatenated string**, no separators:

| Hash | Formula |
|---|---|
| `firsthash` | `MD5(date + filename)` |
| `secondhash` | `MD5(<query1_label><query1_value>...<queryN_label><queryN_value> + date + filename)` |

`date` and `filename` are the same values in both hashes. The query rows are concatenated in the report's `Order` sequence (label = the query's numeric-result placeholder, e.g. `?ReqCountObjectTypes`, value = its result).

This scheme makes the report tamper-evident: changing the report filename, generation date, or any query's numeric result changes `secondhash`.

### Companion tool — `HashVerifier.html`

**Location:** `Code/IC_readertool/HashVerifier.html`

A standalone, offline HTML page (pure JS MD5 implementation, no server calls) that lets a reviewer:

1. Enter the report's date and filename to recompute `firsthash`, and compare it against the value printed in the report footer.
2. Enter the same date/filename plus each query's label/value pair (rows can be added/removed) to recompute `secondhash`.
3. See a ✓ Match / ✗ No match badge against a pasted "expected" hash.

Since it has no dependency on FME or Laces, it can be handed to someone auditing a report without giving them workspace access.

---

## Configuration Workbook

The Excel file (default `IC_Reader_Tool_Config_v5.xlsx`) is the control table for both query execution and report text. Each row defines one validation query:

| Column | Purpose |
|---|---|
| `Query` | The SPARQL query text executed against the uploaded publication. |
| `QueryTitle` | Section heading in the report. |
| `Explanation` | Descriptive text shown alongside the result table. |
| `Order` | Numeric sort key controlling report section order (also drives hash concatenation order). |
| `Category` | `vs1` / `vs2` / `0` (blank treated as `0`) — which `Category` run(s) include this query. |
| `Headers` | Report grouping label; defaults to `Misc` if blank. |
| `Type` | Query/result type metadata. |

---

## Implementation Notes

- The uploaded Laces publication is **temporary**: it exists only for the duration of the run and is deleted (`HTTPCaller_2` DELETE) right after the report is generated.
- `Category = 0` (or blank) means "run all applicable queries"; a query row with `Category = 0` is treated as always-included regardless of the selected run category.
- Python is only used for the Turtle/zip-combining step (`rdflib`); the rest of the workspace is transformer-driven.
- The report is deterministic enough (same IC + same queries + same date → same hashes) to support hash-based change tracking between runs.
- Related files in `Workspaces/IC_Reader_Tool/`: `IC_Reader_Tool.fmw` (earlier baseline), `IC_Reader_Tool_jasper.fmw` (prior iteration), `ICreaderVerifier.fmw` (in-progress verifier workspace), `ShoppingCart.fmw` (unrelated Shopping Cart workspace copied into this folder).
- Added a new Category can be done by inserting a new string value into the Config file, Add that same value to the user parameter category in the FME workspace and the filter will automatically function.
