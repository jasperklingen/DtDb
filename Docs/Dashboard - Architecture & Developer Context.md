**File:** `Code/Dashboard/Dashboard.html`  
**Cache file:** stored in FME Flow shared resources (see constants at top of script)

---

## What it does

A single-file (+ json cache) browser dashboard showing FME Flow job history for the BulkImport and SOV apps. It calls the FME Flow API directly from the browser. Job data is persisted in a JSON cache file on FME Flow's shared resources so page loads are fast and history extends to 10 weeks.

---

## Architecture overview

```
Browser (Dashboard.html)
  │
  ├─ On load ──────► Read jobs-cache.json (FME Flow Resources API)
  │                       │
  │                  ┌────┴────────────────────────────────────┐
  │                  │  cache.jobs   → main job list            │
  │                  │  cache.details → publishedParameters     │
  │                  └────┬────────────────────────────────────┘
  │                       │ pre-populate jobDetailsCache Map
  │
  ├─ Incremental fetch ──► RUNNING + QUEUED: always full fetch (few jobs, always changing)
  │                        SUCCESS + FAILURE + CANCELLED: fetch pages until a known
  │                        terminal job ID is found in the page, then stop
  │
  ├─ Merge ────────────► overlay freshTerminal → overlay freshActive → applyRetention
  │
  └─ Save ─────────────► write jobs-cache.json back to FME Flow Resources
```

---

## Key constants (top of `<script>`)

| Constant                | Purpose                                                      |
| ----------------------- | ------------------------------------------------------------ |
| `PAGE_SIZE`             | Jobs per API request (50)                                    |
| `POLL_MS`               | Auto-refresh interval (300 000 ms = 5 min)                   |
| `CACHE_RETENTION_WEEKS` | How far back to keep jobs (10 weeks)                         |
| `CACHE_BASE_URL`        | FME Flow Resources API base for the cache file               |
| `CACHE_READ_URL`        | Full URL to download the cache                               |
| `CACHE_WRITE_URL`       | Full URL to upload the cache (points to directory, not file) |
| `REPOS_FILTER`          | Which FME repositories to fetch                              |
| `REPO_GROUPS`           | Maps filter-label → one or more actual repo names            |
| `ACTIVE_STATUSES`       | `['running', 'queued']` always fully re-fetched              |
| `TERMINAL_STATUSES`     | `['success', 'failure', 'cancelled']` fetched incrementally  |

---

## Cache file structure

```json
{
  "version": 1,
  "updatedAt": "2026-06-20T10:00:00.000Z",
  "jobs": [ ...FME job objects from /fmeapiv4/jobs... ],
  "details": {
    "112193": { "workspace": "uuid-...", "cartUuid": null },
    "112298": { "workspace": null, "cartUuid": "112193" }
  }
}
```

**`jobs`** the main array used to render the table. Contains all jobs fetched from `/fmeapiv4/jobs` across all repos and statuses, filtered to `CACHE_RETENTION_WEEKS`.

**`details`** a map of `jobId → { workspace, cartUuid }`, populated by individual calls to `/fmeapiv4/jobs/{id}`. These are needed because `publishedParameters` (which holds `REL_WORKSPACE` and `SHOPPINGCART_UUID`) is only returned by the single-job endpoint, not the list endpoint. On load, this map is used to pre-populate the in-memory `jobDetailsCache` so those individual calls are not repeated on every page load.

`cartUuid` being set on a SOV job means it was spawned by a ShoppingCart run. Those jobs are **hidden from the main table** they appear only inside the ShoppingCart row's expand panel.

---

## Incremental fetch logic

The stop condition for terminal statuses is **"stop fetching more pages when the current page contains a job ID already in `cachedTerminalIdSet`"**. This set is built from jobs in the cache that are already in a terminal state.

**Important:** `cachedTerminalIdSet` must NOT include previously-active jobs (RUNNING/QUEUED). If it does, the stop fires when the page contains a previously-running job that just finished, causing that job to be silently dropped, it gets removed from the base (was active) but never added from the terminal fetch (stop fired too early).

This was a real bug. The fix is in `fetchJobs()`:

```javascript
const cachedTerminalIdSet = new Set(
  cached
    .filter(j => !prevActiveIds.has(getField(j, 'id', 'jobId')))  // exclude previously active
    .map(j => getField(j, 'id', 'jobId'))
);
```

There is also a safety net (planned, not yet implemented at time of writing): after the merge, any job ID that was in `prevActiveIds` but is missing from the final `jobMap` should be fetched individually, since it finished while the stop condition fired before reaching it.

---

## ShoppingCart ↔ SOV relationship

ShoppingCart jobs (repo `SOV-ShoppingCart`) spawn one or more SOV child jobs. The link is `SHOPPINGCART_UUID` in the SOV job's published parameters, it contains the parent ShoppingCart job ID.

Flow:
1. `prefetchSovCartUuids()` fetches details for all SOV jobs in the background.
2. Any SOV job with `cartUuid` set is added to `sovChildJobIds` and filtered out of `allJobs`.
3. When a ShoppingCart row is expanded, `loadShoppingCartChildren()` finds all SOV jobs where `details.cartUuid === cartJobId` and renders them in the detail panel.

`fetchAndUpdateWorkspace` deduplicates in-flight requests via a `jobDetailsPending` Map, a second caller for the same job ID while a request is in flight receives the same Promise rather than starting a second request. This matters because `prefetchSovCartUuids` and `loadShoppingCartChildren` can run concurrently for the same new job.

---

## FME Flow API notes

The dashboard uses `/fmeapiv4/` (v4) throughout. Auth header: `Authorization: fmetoken token=<token>`.

For the cache file, the Resources API requires:
- **Read:** `GET /fmeapiv4/resources/connections/{connection}/download?path=/...`
- **Write:** `POST /fmeapiv4/resources/connections/{connection}/upload?path=/dir&overwrite=true`  
  Body must be `multipart/form-data` with field name `files` (plural).  
  The upload URL must point to the **directory**, not the file. The filename comes from the FormData entry. Pointing the URL at the file path causes FME to nest the filename inside itself as a subdirectory (500 error).

---

## Debugging tips

All key in-memory state is accessible from the browser console (F12):

```javascript
allJobs.length                    // how many parent jobs are currently displayed
allJobsWithChildren.length        // total including child jobs
sovChildJobIds.size               // how many SOV jobs are hidden as cart children
jobDetailsCache.get(112193)       // workspace/cartUuid for a specific job
jobDetailsCache.has(112193)       // whether details have been fetched for a job

// Check if a job is being filtered out as a SOV child:
sovChildJobIds.has(116337)        // true = hidden, look for it inside a ShoppingCart panel

// Check if a job exists in the loaded list at all:
allJobsWithChildren.find(j => j.id === 116337)
```

**Job missing from the dashboard?** Work through this checklist:
1. `allJobsWithChildren.find(j => j.id === X)`  is it in memory at all?
   - Not there → it was never fetched or was dropped by the incremental stop. Clear the cache file on FME Flow and reload for a cold-start full fetch.
1. `sovChildJobIds.has(X)`  is it hidden as a ShoppingCart child?
   - True → look for it inside the parent ShoppingCart's expand panel.
1. `jobDetailsCache.get(X)`  does it have a stale `null` details entry?
   - `null` → the detail fetch failed at some point and was cached as a failure. Delete that entry from `cache.details` in the file on FME Flow and reload.
1. Check `engineName` on the job object, if it contains `@` (e.g. `localhost_Engine2@112193`), `isParentJob()` returns false and it is always excluded.

**Cache file issues:** The cache file lives in FME Flow shared resources. Deleting it (or resetting it to `{"version":1,"updatedAt":null,"jobs":[]}`) forces a cold-start full fetch on the next page load. This is also the fix if jobs that recently transitioned from RUNNING to a terminal status went missing — the cold-start fetches all terminal jobs from offset 0 with no stop condition.

---

## Known limitations / future work

- The `details` map in the cache grows indefinitely, expired jobs' detail entries are never pruned. A `saveCache` cleanup pass (remove detail entries for IDs not in `jobs`) would keep the file size in check over time.
- The safety net for missed transitioning jobs (fetch individually any `prevActiveIds` not in the merged result) is designed but not yet implemented.
- `FME_FAILURE` and `ABORTED` statuses exist in the UI filter but are not in `TERMINAL_STATUSES`, so those jobs are not fetched. If they appear in the system, add them to `TERMINAL_STATUSES`.
- Might consider moving the cache to the html file as a JSON object. Especially when dataset is pruned to only contain a few weeks of history. 
- The page has a prerendered html table to fill up the space. Since html page is embedded into FME it reserves the physical length of the page as first rendered. Since the contents of the table are dynamically loaded with API calls the page size will not grow dynamically. The template table is there to ensure the page is rendered correctly.
