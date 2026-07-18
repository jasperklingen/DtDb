# DtDb — Claude Code context

## Project summary

FME workspaces and browser tooling for TenneT's digital-twin data pipeline, maintained by Avineon Tensing.
The main moving part is a browser-side job-monitoring dashboard for FME Flow (`Code/Dashboard/Dashboard.html`) — a single HTML file that calls the FME Flow REST API directly, caches job history as a JSON file on FME Flow shared resources, and updates incrementally. The repo also contains the SOV Shopping Cart entry point and the IC Reader Tool validation workspace.

## Read these docs at session start

- `Docs/Dashboard.md` — full architecture, cache structure, incremental fetch logic, FME API quirks, debugging tips, and open items for the dashboard
- `Docs/ShoppingCart.md` — Shopping Cart pipeline (ShoppingCart.fmw → SOV.fmw) and parameter documentation
- `Docs/IC_Reader_Tool.md` — IC validation workspace, SPARQL pipeline, and hash scheme

## Key facts

- FME Flow runs at `https://tennet.vicrea.nl`; API is `/fmeapiv4/`; auth header is `Authorization: fmetoken token=<token>`
- The dashboard cache file lives in the `FME_SHAREDRESOURCE_DASHBOARD` shared resource connection at `/dashboards/jobs-cache.json`
- Upload to FME Flow Resources API requires `multipart/form-data`, field name `files` (plural), URL pointing to the directory (not the file)
- Active job statuses (`running`, `queued`) are always fully re-fetched; terminal statuses (`success`, `failure`, `cancelled`) use an incremental stop-on-known-ID fetch
- SOV child jobs are linked to their parent ShoppingCart job via `SHOPPINGCART_UUID` in `publishedParameters` — these are filtered out of the main table and shown inside the ShoppingCart row's expand panel
- The `jobDetailsCache` Map (populated from `cache.details` on load) avoids repeated individual job API calls

## User

Jasper — FME developer/admin, minimal JS experience. Prefers direct explanations over extensive background.
