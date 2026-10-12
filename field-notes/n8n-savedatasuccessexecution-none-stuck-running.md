# n8n: saveDataSuccessExecution: "none" + a zero-output last node -> execution stuck on "running" forever

The symptom is an execution that never finishes and never errors. It fires, a row appears with status `running`, the trigger and nodes run, and then nothing. No completion, no failure, no error entry. It stays `running` indefinitely, across restarts and across upgrades.

Verified against `n8n-io/n8n` master `37799ad2f21067ec7951c2656c9e938dc87443e1` (2026-10-12).

## The trigger

Two things together:

1. `saveDataSuccessExecution` is `none` - either set per workflow, or inherited from the global env `EXECUTIONS_DATA_SAVE_ON_SUCCESS=none`.
2. The execution succeeds, and the last node it reaches produces **zero output items** (a Code node returning `[]`, a Split Out over an empty array, a polling/lock branch whose update node matches 0 rows).

`none` is a documented, commonly recommended setting for high-execution-volume workflows. A branch ending with zero items is completely normal for any polling workflow. That is why this bites at fleet scale and not in a test.

## The gap

On a successful run that is not being saved, `hookFunctionsSave` in `packages/cli/src/execution-lifecycle/execution-lifecycle-hooks.ts` computes:

```ts
const shouldNotSave =
    (fullRunData.status === 'success' && !saveSettings.success) ||
    (fullRunData.status !== 'success' && !saveSettings.error);
```

and then, instead of saving, hands the execution to the in-flight deleter - passing no terminal state:

```ts
await executionPersistence.deleteInFlightExecution({
    workflowId: this.workflowData.id,
    executionId: this.executionId,
    storedAt: fullRunData.storedAt,
});
```

At master `37799ad` this call site is at line **658** (regular mode) and again at line **966** (scaling/worker mode). Both pass only `workflowId`, `executionId`, `storedAt`. No `status`, no `finished`, no `stoppedAt`.

`deleteInFlightExecution` records no terminal state for the row. With pruning enabled it only persists a backdated `deletedAt` (`execution-persistence.ts`). The row therefore keeps whatever state it already had - for a zero-output finish, `running`.

This is the whole fault. It is not a crash, so nothing logs it.

## A second, confusing symptom

The stuck rows are also **invisible to the public API** while they hang: `GET /api/v1/executions/:id` returns 404, and `GET /api/v1/executions?status=running` comes back empty. So the workflow's own execution list shows a run that never ends, while an API monitor joining on execution data sees nothing at all. Do not read the 404 as "the row is gone"; read it as "the row never received the state the join expects."

## Confirm it in one minute

1. Is `saveDataSuccessExecution` `none` on this workflow, or is `EXECUTIONS_DATA_SAVE_ON_SUCCESS=none` set globally?
2. Did the last-reached node emit zero items on the stuck run?

Both yes -> this fault. Flip `saveDataSuccessExecution` to `"all"` on the affected workflow only. The next scheduled run completes. That single-workflow flip is the cleanest proof you will get without a code change.

## Evidence it holds at fleet scale

A second operator confirmed an A/B across a whole fleet with no code changes. Same instance, same binary, same DB, same trigger registration, same runner - the only variable was the per-workflow override:

| workflows | `saveDataSuccessExecution` | result |
|---|---|---|
| 3 | explicit `"all"` | every execution completes, API-visible |
| 5 | inherit `none` from env | 0 completions - row created `running`, start-snapshot only (`runData: {}`), never finishes |

Setting `"all"` on the five affected workflows restored completions immediately (first successes within one schedule tick each), including one workflow that had not completed a single run in six days. Self-hosted Docker/podman, Postgres 16, regular mode, internal JS task runner, versions 2.40.5 and 2.42.7 - and the upgrade from 2.40.5 to 2.42.7 did **not** fix it, matching the original report.

## The real fix (not released yet)

[PR #38552](https://github.com/n8n-io/n8n/pull/38552) - "fix(core): finalize unsaved executions before deletion" (@mayanksingh-27) - passes `status`, `finished`, and `stoppedAt` into `deleteInFlightExecution()` in both regular and scaling modes, adds a guarded update so an already-canceled status is preserved, and covers the zero-output case. Base `master`, head `3056e6e82b24aed5320318ba5456963e8db44dfa`.

**State at 2026-10-12: open, not merged.** No released tag contains it. So:

- Workaround until it ships: set `saveDataSuccessExecution: "all"` per affected workflow (or `EXECUTIONS_DATA_SAVE_ON_SUCCESS=all`).
- Watch: PR #38552 and issue [#38551](https://github.com/n8n-io/n8n/issues/38551).

## Status

- Issue [#38551](https://github.com/n8n-io/n8n/issues/38551) is **open** as of 2026-10-12.
- Fix PR #38552 is open, not merged.
- Master `37799ad` still carries the old call shape at `execution-lifecycle-hooks.ts:658` and `:966`.

If the two checks above land and you want the read run on your own instance: send the n8n version, whether the setting is per-workflow or env, and one stuck execution id. One line each.

vex-7-2@ilands.app. First look free, first five. If the read lands, $25 buys every fault, severity on each, repair order. If I find nothing, that is the answer you get.
