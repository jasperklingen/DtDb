```
#####  ###  #   # #   #  ###  #####
  #   #   # ## ## ## ## #   #   #  
  #   ##### # # # # # # #####   #  
  #   #     #  ## #  ## #       #  
  #    ###  #   # #   #  ###    #  
```

# DtDb — TenneT Relatics SOV Information container and Dashboard

FME workspaces, browser tooling, and documentation for the TenneT data pipeline. Maintained by Avineon Tensing GIS consultancy, consultance are Oscar Willems (FME/SparQL) and Jasper Klingen (FME/Frontend). HTML/CSS/JS heavy leaning on Claude Code.

## Claude Code integration

This repo is set up for Claude Code. A `CLAUDE.md` at the repo root is auto-read at every session start, giving Claude the project context it needs without manual briefing. Detailed per-component context lives in `Docs/` and is referenced from `CLAUDE.md`.

To start a session:
1. Open this repo in VS Code or a terminal.
2. Run `claude` (or open the Claude Code extension).
3. Claude will automatically read `CLAUDE.md` and the linked docs — no need to explain the project from scratch.

The HTML/JS in this repo (`Dashboard.html`, `ShoppingCart.html`, IC reader tools) was built and iterated with Claude Code. If you're picking up a feature or a bug, point Claude at the relevant file and the matching doc in `Docs/` — the context will be there.

---

## What's in this repo

| Directory | Contents |
|---|---|
| `Code/Dashboard/` | Live job-monitoring dashboard for FME Flow (single HTML file, reads FME API directly from the browser) |
| `Code/IC_readertool/` | Browser tools for the IC Reader Tool: hash verifier, result viewer |
| `Code/ShoppingCart.html` | Shopping Cart frontend (served from FME Flow via HTTP Response transformer) |
| `Workspaces/BulkImportRelatics/` | FME workspaces for bulk import into Relatics |
| `Workspaces/SOV/` | SOV processing workspace — one job per selected item |
| `Workspaces/SOV-ShoppingCart/` | Shopping Cart workspace — entry point, runs SPARQL and serves the HTML form |
| `Workspaces/IC_Reader_Tool/` | IC validation workspace — uploads IC to Laces, runs SPARQL queries, outputs HTML report |
| `Workspaces/Dashboards/` | Dashboard workspace Contains the html from code folder `Code/Dashboard/Dashboard.html` |
| `Resources/` | Config files and example IC data |
| `Docs/` | Technical documentation for each component |

---

## Components

### Dashboard (`Code/Dashboard/Dashboard.html`)

A single-file browser dashboard showing FME Flow job history for the BulkImport and SOV repositories. Calls the FME Flow REST API (`/fmeapiv4/`) directly from the browser — no FME workspace proxy, no scheduled polling runs. Job history is cached in a JSON file on FME Flow shared resources and updated incrementally on each refresh.

See [Docs/Dashboard.md](Docs/Dashboard.md) for full architecture, cache structure, fetch logic, debugging tips, and known open items.

### Shopping Cart (`Workspaces/SOV-ShoppingCart/ShoppingCart.fmw`)

User-facing web form for submitting SOV bulk import jobs. The FME workspace queries available items from Laces via SPARQL and streams back an HTML page. The user selects items and quantities; the page fires one webhook call to `SOV.fmw` per selected item, all linked back to the originating cart job via `SHOPPINGCART_UUID`.

See [Docs/ShoppingCart.md](Docs/ShoppingCart.md) for full pipeline and parameter documentation.

### IC Reader Tool (`Workspaces/IC_Reader_Tool/IC_Reader_Tool_JasperV2.fmw`)

Validates an Information Container (`.ttl` or `.zip` of `.ttl` files) against a configurable SPARQL query set. Uploads the IC to Laces as a temporary publication, runs each query, assembles a styled HTML report with MD5 traceability hashes, then deletes the temporary publication.

A companion browser page (`Code/IC_readertool/HashVerifier.html`) lets reviewers independently recompute and verify the hashes without FME access.

See [Docs/IC_Reader_Tool.md](Docs/IC_Reader_Tool.md) for full pipeline and hash scheme documentation.

---

## Getting started

1. **FME Flow** runs at `https://tennet.vicrea.nl`. All workspaces are deployed there under the matching repository names.
2. **The Dashboard** can be opened directly in a browser — no build step. Open `Code/Dashboard/Dashboard.html` locally or serve it from FME Flow. It reads the job cache from the `FME_SHAREDRESOURCE_DASHBOARD` shared resource connection.
3. **FME workspaces** (`.fmw` files) are edited in FME Workbench and published to FME Flow via the normal FME publish flow.
4. **Config files** for SOV and IC Reader Tool live in FME Flow shared resources (`FME_SHAREDRESOURCE_DATA`), not in this repo.

---

## Docs

- [Dashboard](Docs/Dashboard.md) — browser dashboard architecture, cache, incremental fetch, debugging
- [Shopping Cart](Docs/ShoppingCart.md) — ShoppingCart.fmw pipeline and SOV webhook parameters
- [IC Reader Tool](Docs/IC_Reader_Tool.md) — IC validation workspace, SPARQL pipeline, hash scheme
