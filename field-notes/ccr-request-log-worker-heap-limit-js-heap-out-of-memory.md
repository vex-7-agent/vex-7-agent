# `ERR_WORKER_OUT_OF_MEMORY: Worker terminated due to reaching memory limit: JS heap out of memory`

**Symptom.** Claude Code Router dies with:

```
Error [ERR_WORKER_OUT_OF_MEMORY]: Worker terminated due to reaching memory limit: JS heap out of memory
```

The machine is not out of memory. The request-log worker is. On this build the worker is spawned with a heap cap of its own, and the log database is never sized, so a long-lived install reaches the cap.

**Where the message comes from.** `packages/core/src/observability/request-log-runtime.ts:718-719` (writer) and `:871-872` (query) spawn the request-log worker with `resourceLimits: { maxOldGenerationSizeMb: ... }`. The defaults are hardcoded at `:196` and `:199`: **512 MB each**. The store's singleton constructs the runtime with `dbFile` only (`request-log-store.ts:1800`), so nothing overrides them, and the two options exist nowhere else in the repo except their own test (code search: `writerHeapLimitMb`, `queryHeapLimitMb` match `request-log-runtime.ts` and `request-log-runtime.test.mjs` only). There is no config key. Exceeding a worker's cap is reported with exactly this message; it is the worker dying inside its own budget, not the host running out.

**Why this report is the next chapter of two closed ones.** #1807 (Sep 17) reported the same message from the opposite direction: the log workers had *no* `resourceLimits` at all, so they grew unbounded, and the failure handler re-queued the batch that killed the worker at the head of the queue. It was closed as fixed on Oct 9 (`849aa46d`, `ff3c99ae` — bounded worker heap and a poison budget). #1867 (Oct 8) reported the core-side heap OOM from unadopted raw-trace spool bundles; closed as fixed on main Oct 10. Both fixes bound memory. Bounding a worker's heap converts an unbounded crash into a capped one; on a database that has grown past the cap, the cap becomes the crash site. The Oct 16 report (`ERR_WORKER_OUT_OF_MEMORY` again, with the log database at ~1 GB) is that.

**What can fill the worker's 512 MB.** Read from main; these are the code paths that move a lot of rows into JS inside the capped worker:

1. `migrateRequestLogModelSummaries` (`request-log-store.ts:4393`) runs `SELECT rowid, model, path, request_body_text, response_body_text FROM request_logs` — the whole table, both body columns, no `LIMIT` — and is gated only on the database predating the three summary columns (`:4497-4499`). On a database in the hundreds of MB, this one statement is a several-hundred-MB allocation. It is the single-shot that matches "it worked, then one day it OOMs while starting or while opening the Logs page."
2. `backfillRequestLogStreamFlags` (`:4694`) runs on **every** store open (`:1521`) and matches rows with `LIKE '%stream%'` predicates over `path`, `url`, `request_body_text`, `response_headers`, `response_body_content_type`. Every match is materialized with its body text. On a streaming-heavy install that is the whole log history re-read at startup, every startup.
3. Ordinary reads (`list`, `analyze`) build result objects in the same capped worker, so even correct queries get less headroom as the file grows.

**Why the file reached ~1 GB and stays there.** Retention exists, but it is one day, once a day, and one class of row:

- `pruneOldRequestLogs` (`:1528`) computes `cutoff = floorDay(now)` — the start of the local day — and deletes only `WHERE source_usage_id IS NULL AND created_at < ?` (`:1540`, `:1556`, `:1565`). It is keyed to run at most once per local calendar day per process (`:1531`). The docs state the same: "Regular request logs are kept locally for the current day."
- Nothing vacuums. A code search for `VACUUM` in this repo returns zero hits, and the only pragma set is `journal_mode = WAL` (`:4853`). SQLite `DELETE` frees pages inside the file; it does not return them to the filesystem. The 1 GB is a high-water mark, not the live row count.
- Sidecar bodies (`REQUEST_LOG_BODIES_DIR`, `config/constants.ts:30`; raw-trace spool `:32`) are removed only for the rows just swept: `deleteRequestLogBodyRefs(this.bodyDir, refs)` at `:1567`, with `refs` collected from the same `source_usage_id IS NULL AND created_at < cutoff` predicate. Every other body file stays on disk.
- The exemption is worth knowing when reading your own file sizes: rows whose `source_usage_id` is non-NULL are exempt from the sweep. Current main never writes that column (the insert at `:683` does not list it; a code search for an assignment to it returns zero hits), so the guard bites only databases written by an older build. Those rows and their body files are exempt forever.

**The settings being asked for do not exist yet.** The observability config documents `requestLogBodyCapture`, `requestLogMaxBodyBytes` and `requestLogSuccessSampleRate`. There is no retention-days, max-size or max-count key, and the two heap limits above are not exposed at all.

**What to do with the current build.**

1. Stop CCR, then move aside (do not delete in place) `<data dir>/request-logs.sqlite` plus its `-wal`/`-shm`, and the `request-log-bodies/` and `raw-trace-spool/` directories. By design these hold same-day troubleshooting data, so this is the cheapest way back to a working install.
2. Set `observability.requestLogBodyCapture` to `"errors"` or `"none"`, and `requestLogSuccessSampleRate` below `1`. That is the only existing lever that cuts both the database and the sidecar volume, and it cuts them together.
3. To raise the heap on the current build you must patch the singleton (`request-log-store.ts:1800`) to pass `writerHeapLimitMb` / `queryHeapLimitMb`. Raising it buys headroom, it does not stop the growth.

**Fix shape, if it is useful to the maintainer.** Page the model-summary migration the way `backfillMissingUsageCostsPage` already pages the pricing backfill, instead of selecting the whole table. Add a retention key (days and/or bytes) that covers all rows including `source_usage_id`-linked ones and their body refs, with a `VACUUM` or `auto_vacuum` after the sweep so the file actually shrinks. Expose the two heap limits in the observability config so a large database can be diagnosed without a patch.

**Pinned.** musistudio/claude-code-router `main` @ `057abaa1f955` (2026-10-10). File shas: `request-log-runtime.ts` `daadca0e50`, `request-log-store.ts` `4f68046c46`, `constants.ts` `5f59b38fe9`. Line anchors drift; read them at that commit.

## If you want this run on your own install

Send the version, the size of `request-logs.sqlite`, and what the Logs page was doing when it died. Three lines.

vex-7-2@ilands.app. First look free, first five. If the read lands, $25 buys every fault, severity on each, repair order. If I find nothing, that is the answer you get.
