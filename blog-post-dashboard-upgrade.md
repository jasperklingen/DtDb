# From FME Workspace to Live Browser Dashboard — With a Little Help from AI

*How I rebuilt a job-history dashboard without really knowing JavaScript*

---

## The Old Setup

If you've built web apps inside FME Flow, you probably know the pattern: an FME workspace runs on a schedule, calls the FME REST API using HTTPCaller, massages the data, and writes the result to a static HTML file or JSON that a dashboard page reads on load.

That's exactly where this started. We had a dashboard for two of our apps — BulkImport and SOV — that our users checked daily to see which jobs ran, who triggered them, and whether they succeeded. The architecture was:

1. An FME workspace scheduled every **5 minutes**
2. HTTPCaller fetching jobs from the FME Flow API for each repo/status combination
3. The results published directly into an HTML page as inline data
4. The page refreshed itself every 60 seconds

It worked. But it had real limitations:

- **Job log clutter.** Every 5-minute workspace run showed up in the job history, drowning out the actual app runs. At scale, that's ~288 noise entries per day.
- **Staleness.** The data was always between 0 and 5 minutes old — and the workspace had to actually succeed for it to update at all.
- **Hard retention cap.** The workspace fetched at most 500 jobs per status/repo combination in one pass. We wanted to extend history from 4 weeks to 10 weeks. That meant more jobs, more HTTP calls, more workspace runtime.
- **Static.** You could filter, but there was no sorting, no drill-down, no pagination. The table just showed everything.

The workspace itself wasn't doing anything clever — it was essentially acting as an authenticated proxy and a scheduled fetch. We started asking: why not have the browser do this directly?

---

## The Idea

FME Flow's API is perfectly reachable from a browser. The dashboard HTML already had a hardcoded token (internal network, controlled environment). So in theory you could skip the FME workspace entirely and have the JavaScript fetch jobs directly on page load.

The catch: I'm an FME developer and admin. I write workspaces, not JavaScript. I can read JS and tweak things, but building something like incremental pagination, a persistent cache, deduplication logic, and async merges from scratch? That's not where I live.

This is where AI came in — specifically Claude Code — and it's the part I want to be upfront about, because I think there's a useful story here for people in similar situations.

---

## The Increments

### Increment 1: Direct API calls from the browser

The first step was simply moving the fetch logic out of the FME workspace and into the HTML page. The browser calls the FME Flow API directly, one request per repo/status combination, renders the result. The FME workspace is gone entirely.

This immediately solved the job log clutter. But we were still limited to 50 jobs per request, and paging was broken — the original code used a single global `offset` across all 12 parallel requests (3 repos × 4 statuses), which meant paging forward for one status would also skip pages for all the others. Jobs were silently missed.

**The fix:** each repo/status pair now tracks its own offset and stops independently when the API returns fewer than PAGE_SIZE items. All 12 pairs run in parallel per page, and the loop only ends when every pair is exhausted.

### Increment 2: Client-side caching with a JSON file

Fixing the pagination meant the cold-start case — loading 10 weeks of history — could hit 50+ API calls. That's fast but noisy on the network tab, and it happens on every single page load.

The solution was a JSON cache file stored in FME Flow's shared resources. The browser reads `jobs-cache.json` on load, and writes it back after every refresh. The FME workspace is completely gone — no schedule, no job logs.

This took some API archaeology. FME Flow's v3 Resources API returned 403 for our user (permission model), and the v4 endpoint had different requirements — it needed `multipart/form-data` with the field named `files` (plural), and the upload URL had to point to the **directory**, not the file path, otherwise FME would nest the filename inside itself as a subdirectory. None of that is well documented. We found it by reading error messages and cross-referencing with what little v4 documentation exists.

### Increment 3: Incremental updates — stop when you hit the cache

A full-replace cache is fine, but it still fetches everything on every refresh. Since terminal jobs (SUCCESS, FAILURE, CANCELLED) never change their data, there's no reason to re-fetch them.

The new logic:
- **Active statuses** (RUNNING, QUEUED): always fetch all pages. There are few of them and they change constantly.
- **Terminal statuses** (SUCCESS, FAILURE, CANCELLED): fetch pages until a page contains a job ID already in the cache — then stop. We've reached known territory.
- On merge: replace old active entries, overlay new terminal entries (which also handles status transitions like RUNNING → SUCCESS), and prune anything older than the retention window.

FME job IDs are monotonically increasing integers, so "hit a known ID" is a reliable stop condition even without a date filter parameter in the API.

### Increment 4: Persisting job detail lookups

Each SOV job requires a second API call to fetch its `publishedParameters` — specifically the `RelaticsWorkspace` UUID and `SHOPPINGCART_UUID`. On the first load these are fetched lazily as jobs scroll into view. But on every subsequent page load, all ~120 of those detail calls fired again from scratch.

The fix was to include the detail cache in the JSON file alongside the job list. On load, the in-memory `jobDetailsCache` Map is pre-populated from the file. `fetchAndUpdateWorkspace` already had an early-return for cache hits — so after the first full load, zero detail API calls happen on page reload.

This also fixed a race condition: when a new ShoppingCart job appeared, both `prefetchSovCartUuids` and `loadShoppingCartChildren` would simultaneously call `fetchAndUpdateWorkspace` for the same child job IDs, each finding the cache empty, each firing a duplicate request. The solution was a second Map (`jobDetailsPending`) that holds in-flight Promises — a second caller for the same job ID receives the existing Promise instead of starting a new request.

---

## Where AI Actually Helped

I want to be honest about this because I've seen blog posts that mention AI like a footnote. Here's where it actually made the difference:

**Designing the data flow.** I could describe what I wanted in plain language — "stop fetching terminal jobs when you hit an ID that's already in the cache, but always fetch active ones fully" — and get back working code with the edge cases already handled (status transitions, race conditions, cold-start behavior).

**JavaScript patterns I wouldn't have reached for.** The in-flight Promise deduplication pattern (storing Promises in a Map to avoid parallel duplicate requests) is something I would not have written myself. Neither is the per-task offset tracking in the parallel fetch loop.

**API debugging.** When we got a 415, a 403, and a 500 back-to-back from FME's resources API, working through what each error meant and what to change was faster with AI than with trial and error alone. The 500 error message — `jobs-cache.json\jobs-cache.json` — immediately told us the filename was being doubled. Recognizing that as a "URL points to the file, form field also carries the filename" problem is the kind of thing that's obvious once someone points it out.

**Things I could verify myself.** Because the logic was always explained, not just generated, I could trace it and understand why. When something behaved unexpectedly, I could reason about it. That matters — this is running in production.

---

## What the Dashboard Does Now

Compared to the original flat static table:

| Feature | Before | After |
|---|---|---|
| Data source | FME workspace (scheduled) | Browser calls API directly |
| Retention | ~4 weeks (500 job cap) | 10 weeks, configurable |
| Job log clutter | ~288 workspace runs/day | Zero |
| Page load | Re-fetch everything | Read cache, ~12 API calls for incremental update |
| Detail API calls on reload | 120 per load | 0 (pre-populated from cache file) |
| Sorting | None | All columns, click to toggle |
| Pagination | None (all rows at once) | Client-side, 50 rows/page |
| Filters | Client-side flat filter | All 4 filters client-side against full cached dataset |
| ShoppingCart drill-down | Not present | Expandable panel showing linked SOV child runs |
| Duplicate requests | Not applicable | Deduplicated via pending Promise map |

---

## Takeaway for FME Developers

You don't have to be a JavaScript developer to build something like this. The FME Flow API is well-structured, the HTML/JS combination gives you a lot for free, and AI can bridge the gap between "I know what I want" and "I know how to implement it in JS."

What helped most was being specific about requirements and being willing to iterate. "Stop fetching when you hit a cached ID" is a clear enough instruction that the implementation follows naturally. The domain knowledge — knowing that FME job IDs increment, that terminal jobs don't change, that ShoppingCart runs spawn child SOV jobs — that came from knowing the platform. The JavaScript to act on that knowledge came from AI.

The code lives in the dashboard HTML as a single self-contained file with no build step, no dependencies beyond Bootstrap and vanilla JS. Any FME developer who can read it can maintain it.

---

*If you're doing something similar with FME Flow's API — building custom dashboards, integrating job history into other tools, or just trying to get out of the "scheduled workspace as a data proxy" pattern — feel free to reach out.*
