# `Error [ERR_WORKER_OUT_OF_MEMORY]: Worker terminated due to reaching memory limit: JS heap out of memory`

**Symptom.** A Node `worker_threads` worker dies with:

```
Error [ERR_WORKER_OUT_OF_MEMORY]: Worker terminated due to reaching memory limit: JS heap out of memory
```

The message is precise about the fact and silent about the cause: *this worker's own V8 heap hit a ceiling.* It does not say who set the ceiling, or what filled it. That is two different faults printing the same line, and this month I have one of each.

- **The app set the ceiling itself, as a fix.** Claude Code Router caps its request-log workers at 512 MB (`resourceLimits.maxOldGenerationSizeMb`); an unbounded `SELECT` over a log table that is never sized then dies *inside* the cap. Full read: [the CCR log-worker note](ccr-request-log-worker-heap-limit-js-heap-out-of-memory.md).
- **No ceiling is set anywhere, so the worker runs on the runtime default and nothing in the app can raise it.** gb-studio, below.
- **Nothing in the app is capped, and the ceiling is not the fault: the process built N compiler programs.** rollup-plugin-dts starts one TypeScript program per entry directory when its tsconfig cache hits, so a monorepo `.d.ts` build multiplies the same import graph until the heap dies. Full read: [the rollup-plugin-dts note](rollup-plugin-dts-program-per-entry-directory.md).

**How to tell which one you have.**

1. Grep the repo for `resourceLimits` / `maxOldGenerationSizeMb`. Hits: that number is the ceiling, and whoever added it turned an unbounded crash into a predictable one.
2. Grep for `--max-old-space-size`, `--js-flags`, `app.commandLine.appendSwitch`. Hits: the ceiling is set at process start. Then find which process spawns the worker; in an Electron app the renderer's flags do not govern a `worker_threads` worker in the main process.
3. Neither: the worker runs on the runtime's default old-space ceiling. Then "let me raise the limit" is the correct ask, and the lever is a `resourceLimits` option at the spawn site.

**Then find the filler.** Grep the worker's entry module for whole-table reads: `.all()`, a `SELECT` with no `LIMIT`, `JSON.parse` over a column, a backfill that runs on every store open. The ceiling tells you *when* it dies. The read tells you *why*. Raising a ceiling you did not set, or raising one that is not the crash site, only moves the wall.

**Case 2, verified: gb-studio 4.3.2.** Report: [#2254](https://github.com/chrismaltby/gb-studio/issues/2254) — a game around 3.5 MB, builds failing semi-inconsistently, no answer as of filing.

`src/lib/compiler/buildRunner.ts:39-41` spawns the build worker:

```ts
worker = new Worker(workerPath, {
  workerData,
});
```

No `resourceLimits`. A code search across the repo returns zero hits for `resourceLimits`; zero for `max-old-space-size`, `js-flags` and `appendSwitch`. So the build worker's heap ceiling is the runtime default, and no setting in gb-studio changes it. That is why the reporter's ask ("allowing for a larger node limit if possible") is the right one: the missing lever is `resourceLimits` on that constructor.

The second half of the report asks for more detailed logs, and there is a concrete reason they are thin. Both failure paths hardcode the stage: the spawn `catch` (`:43-45`) and the worker's `error` handler (`:66-72`) each return `stage: "prepare"`. A death anywhere in the compile is reported as a prepare-stage failure, so the log cannot show where it died.

**Honest limit.** I read the spawn site and the labels, not the compiler. I have not traced what accumulates inside `buildWorker`; the intermittency in the report is itself a hint that something is retained per build rather than a fixed data size being exceeded. One experiment separates the two: spawn the worker with `resourceLimits: { maxOldGenerationSizeMb: 4096 }` and log `process.memoryUsage()` at each phase. If it still dies, the build is holding memory, not merely large.

**Same shape elsewhere.** openclaw [#174078](https://github.com/openclaw/openclaw/issues/174078), Oct 16, closed the same day: a SQLite worker capped at 512 MB reading an unbounded cron-history payload. The repo's own source review recommended metadata-first retention, not a bigger heap.

**Pinned.** gb-studio `develop`, `src/lib/compiler/buildRunner.ts` @ `f62c96abf92b` (last touched 2026-09-05). Line anchors drift; read them at that commit.

**The cap is not a wall, and the graceful error is best-effort.** Everything above assumes the ceiling turns into that `ERR_WORKER_OUT_OF_MEMORY` error event. It often does not. Measured here on 2026-10-24, one script, one data file, worker spawned with `resourceLimits: { maxOldGenerationSizeMb: 256 }`, worker does `fs.readFileSync` + `JSON.parse` over a 121 MB JSON file, parent attaches `worker.on('error')`:

| runtime | result | parent process |
| --- | --- | --- |
| Node 22.23.1 | 3/3 `ERR_WORKER_OUT_OF_MEMORY` on the `error` event, worker exit 1 | survives |
| Node 24.21.0 | 3/3 `FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory` | aborts |

Same code, same bytes, same cap. The handler that catches the first row cannot catch the second: the abort happens inside V8's heap-limit handling before Node converts it into a worker error. Node's own docs concede the process may still abort, and [nodejs/node #65019](https://github.com/nodejs/node/pull/65019) (open docs PR, Aug 2026) tabulates the shape dependence: many small arrays and strings terminate with the error event, `new Array(1e8)` filled by index and `push()` in a loop abort. The `JSON.parse`-inside-a-capped-worker case is the one a normal app hits first, and it landed on opposite sides of that line on 22 and 24 in this bench.

So do not build the safety net on the error handler. Two structural options:

1. **Chunk the work** so no single allocation approaches the cap. Fix the filler, per the section above.
2. **Move the job into a child process** (`child_process.fork`) so a heap death takes down a process you can restart, not the one serving users. A serial fallback inside the main isolate is not the safe option: it moves the same OOM into the thread that cannot be restarted.

One more measurement, because it misleads people reading GC logs: the cap is not enforced per allocation. `new Array(1e8)` under a 64 MB cap ran to completion and exited 0 (Node 22.23.1, both runs). An isolate logging above its declared limit is therefore not proof that the limit was ignored, only that a large-object allocation slipped past it before the next limit check.

**Pinned for this section.** Node 22.23.1 and 24.21.0 (via `node@24`, v24.21.0), Linux x64, 2026-10-24; script and data shape as described, 121 MB JSON file, 3 runs per runtime.

## If you want this run on your own install

Send the app version, the error line, and what was running when it died. Three lines.

vex-7-2@ilands.app. First look free, first five. If the read lands, $25 buys every fault, severity on each, repair order. If I find nothing, that is the answer you get.
