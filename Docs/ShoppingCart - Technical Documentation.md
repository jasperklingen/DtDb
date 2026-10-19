## Overview

The Shopping Cart is a user-facing web form for running SOV workspace based on items selected from a standard design. It works as a two-stage pipeline:

1. **`ShoppingCart.fmw`** queries available items via SPARQL and streams an HTML page back to the user, with the query results embedded as CSV.
2. **`ShoppingCart.html`** renders those results as an interactive tree, lets the user select items and quantities, and submits **one `SOV.fmw` job per copy** through the FME Flow REST jobs API.

![[ShoppingCart-System Overview.drawio.png]]

---

## Components

### 1. `ShoppingCart.fmw` — FME Workspace

**Location:** `Workspaces/SOV_Shoppingcart/ShoppingCart.fmw`

**Responsibility:** Entry point. Accepts parameters, runs a SPARQL query to retrieve available options, then serves the HTML page with the query results injected.

**Note:** This process is an artifact of earlier solutions, there is no particular reason to make a CSV. This was just the easiest way to transfer the data from FME to HTML.
#### Published Parameters

| Parameter                 | Description                                                                                                                                         |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CONFIG`                  | Name of the config Excel file (without extension), e.g. `SOV`. The full path is resolved to `$(FME_SHAREDRESOURCE_DATA)/SOV/Config/$(CONFIG).xlsx`. |
| `RelaticsWorkspace`       | UUID, Target Relatics workspace identifier. Passed through to `SOV.fmw`.                                                                            |
| `BatchSize`               | Number of items processed per batch in `SOV.fmw`. Passed through unchanged.                                                                         |
| `MaxJobRuntime`           | Maximum runtime in seconds for each child `SOV.fmw` job. `0` (or empty/invalid) means no limit. Passed to the REST API as `maxJobRuntime`.          |
| `FME_SERVER_TOKEN`        | FME Server token injected automatically. Used by the HTML to authenticate job submissions.                                                          |
| `FME_JOB_ID`              | FME Server job ID injected automatically. Used as `SHOPPINGCART_UUID` to correlate all child jobs back to this cart run.                            |
| `FME_SHAREDRESOURCE_DATA` | FME shared resource path, injected automatically. Used to resolve the config file path.                                                             |

#### Transformer Pipeline

The workspace processes data through six logical stages:

**1. Config loading**

The Excel config file (`SOV.xlsx`, sheet `SOVQuerys`) is read. A `Tester` + `Sampler` + `AttributeKeeper` combination filters the row where `QueryName = "Tree"` and retains only the `Query` and `QueryEndpoint` columns.

**2. SPARQL query execution**

`AC_Query` writes the SPARQL query string to a feature attribute. An `HTTPCaller` then POSTs it to the `QueryEndpoint` using the "TenneT Laces" web connection. The raw JSON response is stored in `_response_body`; the HTTP status code in `_http_status_code`.

**3. Response parsing**

A `JSONFragmenter` flattens the JSON response at `bindings[*]`, producing one feature per result row. Each feature carries attributes for the child node, parent node, names, and types. `AttributeExposer` + `BulkAttributeRenamer` normalise the attribute names to the expected schema.

**4. Tree construction**

An `InlineQuerier` runs a recursive SQL query (`WITH RECURSIVE`) over the parent–child relations to derive a full tree structure, including `depth` and `path` for each node. A `StringConcatenator` formats each node as a single CSV line (`uuid,path`). A `Sorter` + `Aggregator` then concatenates all lines into one CSV string stored in `_result`.

**5. HTML assembly**

`AC_header_CSV` prepends the header row (`child,path`) to the CSV string. `AC_HTML_JSPagina` injects the complete CSV into the `ShoppingCart.html` template via the `@Value(csv)` placeholder.

**6. HTTP response**

The HTML Writer (format: HTML, dataset: `Tree.html`) is registered on the FME Data Streaming service. Instead of writing to disk, it streams the assembled HTML page directly back to the user's browser as the HTTP response.

---

### 2. `ShoppingCart.html` — Frontend

**Location:** `SOV_Shoppingcart/ShoppingCart.html`

**Responsibility:** Rendered in the user's browser. Parses the injected CSV, builds an interactive tree, collects selections, and submits `SOV.fmw` jobs through the FME Flow REST API.

#### Data Injection

The CSV from `ShoppingCart.fmw` is embedded directly into the HTML at load time:

```js
const csvData = `@Value(csv)`;
```

FME replaces `@Value(csv)` with the actual CSV string before streaming the response.

#### CSV Format

```
uuid,path
3f2a1b...,Hoofdgroep|Subgroep|Itemname
```

- **Column 1:** UUID — used as `SELECTED_GROUP` in the submitted `SOV.fmw` job.
- **Column 2:** Pipe-delimited path — determines the tree hierarchy rendered in the UI.

#### Tree Rendering

- Paths are split on `|` and built into a nested tree structure.
- Top-level nodes are expanded by default; deeper levels are collapsed.
- Each node has a quantity input (integer, 0–20).
- Nodes without a UUID (intermediate grouping nodes) can still have a quantity assigned. These are stored as `path:<path>` in the input's `data-node-id` and are submitted as-is, so the SOV workspace must tolerate a non-UUID `SELECTED_GROUP` (or those nodes should be left at 0).

#### Submit Behaviour

On clicking **Run**:

1. All nodes with quantity > 0 are collected as `{ id, label, count }`.
2. For each selected node, **one `SOV.fmw` job is submitted per copy** (`count` jobs), each with `COPIES = '1'`. This replaces the older approach of passing the total quantity in `COPIES` to a single streaming GET call.
3. Jobs are submitted with `POST https://tennet.vicrea.nl/fmeapiv4/jobs` and the JSON body below.
4. Submissions are fire-and-forget: responses are not awaited, and only failures are logged to the browser console (`console.warn`).
5. A timestamped batch header plus one result row per selected item (`<n> job(s) sent`) is appended to the log at the bottom of the page.
6. All quantity inputs are reset to 0 after submission.

#### Job Submission Request

```http
POST /fmeapiv4/jobs
Authorization: fmetoken token=<FME_SERVER_TOKEN>
Content-Type: application/json
```

```json
{
  "repository": "SOV",
  "workspace": "SOV.fmw",
  "publishedParameters": {
    "CONFIG_XLSX": "<FME_SHAREDRESOURCE_DATA>/SOV/Config/<CONFIG>.xlsx",
    "SELECTED_GROUP": "<uuid or path:...>",
    "SELECTED_GROUP_NAME": "<Itemname>",
    "COPIES": "1",
    "HubLaces": "",
    "RelaticsWorkspace": "<RelaticsWorkspace>",
    "BATCH_SIZE": "<BatchSize>",
    "SHOPPINGCART_UUID": "<FME_JOB_ID>"
  },
  "maxJobRuntime": 1
}
```

`maxJobRuntime` is only included when `MaxJobRuntime > 0`.

---

### 3. `SOV.fmw` — Downstream Workspace

**Location:** FME Server, repository `SOV`, workspace `SOV.fmw`

**Responsibility:** Processes one selected item per invocation. Triggered by the Shopping Cart HTML through the REST jobs API, once per selected copy. `COPIES` is always `1`; the number of copies is expressed as the number of jobs submitted.

#### Parameters Received

| Parameter | Source | Description |
|---|---|---|
| `CONFIG_XLSX` | `$(CONFIG)` from ShoppingCart | Full path to the config Excel file. |
| `SELECTED_GROUP` | Node UUID from SPARQL result | Identifies which item to process. |
| `SELECTED_GROUP_NAME` | Node name from SPARQL result | Human-readable label of the selected node (e.g. `Itemname`). Required in `SOV.fmw`; used by the Dashboard to show which item a job processed. |
| `COPIES` | Always `1` | Copies per job; multiplicity is handled by submitting multiple jobs. |
| `HubLaces` | Hard-coded empty string | Reserved; currently unused. |
| `RelaticsWorkspace` | Pass-through from ShoppingCart | Target Relatics workspace. |
| `BATCH_SIZE` | Pass-through from ShoppingCart | Batch size for processing. |
| `SHOPPINGCART_UUID` | `$(FME_JOB_ID)` from ShoppingCart | Groups all child jobs under the originating cart run. |

#### Job Runtime Limit

`maxJobRuntime` is set on the submitted job when `MaxJobRuntime > 0`, giving FME Flow a hard ceiling on child job duration.

#### `SELECTED_GROUP_NAME`

The HTML sends `SELECTED_GROUP_NAME`, the display name of the selected node (e.g. `Itemname`, without parent path). It is **not used by the SOV workspace logic** — it exists so the Dashboard can display a readable item name next to each child job instead of only the UUID.

Add it to `SOV.fmw` as an optional published parameter:

```
GUI_LINE="GUI OPTIONAL STRING SELECTED_GROUP_NAME Selected Group Name"
DEFAULT_VALUE=""
IS_STAND_ALONE="true"
```

It must stay **optional**: cart jobs always supply a value, but direct runs from the FME web UI or Workbench do not. No `INCLUDE [...]` validation line is needed, and no transformer wiring is required since the value is read from the job record's `publishedParameters`.

Only jobs submitted after this parameter was introduced carry the name. Older jobs show `–` in the **Groep** column; there is no backfill.

---

## Configuration File (`$(CONFIG).xlsx`)

The config file is an Excel workbook stored at:

```
$(FME_SHAREDRESOURCE_DATA)/SOV/Config/$(CONFIG).xlsx
```

Its name is set via the `CONFIG` published parameter and is shown in the Shopping Cart page header. The config file drives the SOV workspace behaviour (mapping rules, field definitions, etc.) and is not read by the Shopping Cart itself.

---

## Adding or Changing Available Items

This is out of scope for the developer. This is purely a functional admin resposibility. Any changes should be made in the config excel or standard design.

---

## Deployment

| Artifact | Location on FME Server |
|---|---|
| `ShoppingCart.fmw` | `IC_Reader_Tool/ShoppingCart.fmw` |
| `ShoppingCart.html` | Embedded inside `ShoppingCart.fmw` (HTTP Response transformer) |
| `SOV.fmw` | `SOV/SOV.fmw` |
| Config Excel | `$(FME_SHAREDRESOURCE_DATA)/SOV/Config/<CONFIG>.xlsx` |

