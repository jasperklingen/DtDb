## Overview

The Shopping Cart is a user-facing web form for running SOV workspace based on items selected from a standard design. It works as a two-stage pipeline:

1. **`ShoppingCart.fmw`** queries available items via SPARQL and streams an HTML page back to the user, with the query results embedded as CSV.
2. **`ShoppingCart.html`** renders those results as an interactive tree, lets the user select items and quantities, and fires a webhook call to **`SOV.fmw`** for each selected item and quantity.

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
| `FME_SERVER_TOKEN`        | FME Server token injected automatically. Used by the HTML to authenticate webhook calls.                                                            |
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

**Responsibility:** Rendered in the user's browser. Parses the injected CSV, builds an interactive tree, collects selections, and makes webhook calls.

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

- **Column 1:** UUID — used as `SELECTED_GROUP` in the SOV webhook URL.
- **Column 2:** Pipe-delimited path — determines the tree hierarchy rendered in the UI.

#### Tree Rendering

- Paths are split on `|` and built into a nested tree structure.
- Top-level nodes are expanded by default; deeper levels are collapsed.
- Each node has a quantity input (integer, 0–20).
- Nodes without a UUID (intermediate grouping nodes) can still have a quantity assigned.

#### Submit Behaviour

On clicking **Run**:

1. All nodes with quantity > 0 are collected.
2. For each selected node, a `GET` request is fired to `SOV.fmw` with the parameters below.
3. Requests are fire-and-forget (no response awaited). All the requests that are made each run are shown in the ordered list at the bottom of the page.

---

### 3. `SOV.fmw` — Downstream Workspace

**Location:** FME Server at `https://tennet.vicrea.nl/fmedatastreaming/SOV/SOV.fmw`

**Responsibility:** Processes one selected item per invocation. Triggered by the Shopping Cart HTML, once per selected node. Amount selected on HTML page is passed on as COPIES parameter.

#### Webhook URL Structure

```
GET /fmedatastreaming/SOV/SOV.fmw
    ?CONFIG_XLSX=<FME_SHAREDRESOURCE_DATA>/SOV/Config/<CONFIG>.xlsx
    &SELECTED_GROUP=<uuid>
    &COPIES=<quantity>
    &RelaticsWorkspace=<RelaticsWorkspace>
    &BATCH_SIZE=<BatchSize>
    &SHOPPINGCART_UUID=<FME_JOB_ID>
```

**Authorization:** `Authorization: fmetoken token=<FME_SERVER_TOKEN>` (header).

#### Parameters Received

| Parameter | Source | Description |
|---|---|---|
| `CONFIG_XLSX` | `$(CONFIG)` from ShoppingCart | Full path to the config Excel file. |
| `SELECTED_GROUP` | Node UUID from SPARQL result | Identifies which item to process. |
| `COPIES` | User input (1–20) | How many copies/instances to import. |
| `RelaticsWorkspace` | Pass-through from ShoppingCart | Target Relatics workspace. |
| `BATCH_SIZE` | Pass-through from ShoppingCart | Batch size for processing. |
| `SHOPPINGCART_UUID` | `$(FME_JOB_ID)` from ShoppingCart | Groups all child jobs under the originating cart run. |

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

