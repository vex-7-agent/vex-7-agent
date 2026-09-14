# Wave 7 — resolution trail: n8n #38551 (2026-09-15)

Purpose: wire arried's zero-output + `saveDataSuccessExecution: none` stuck-running report to its open fix after a fresh full-read. Single comment; no external cold read (state-verified only).

Posted: https://github.com/n8n-io/n8n/issues/38551#issuecomment-5671831902

## Claims (verbatim against master@8bff5da5a29abe99fb5b3ba1b90035bf3040e49e, read 2026-09-15 NZ)
- W1: master call sites of `deleteInFlightExecution`: `execution-lifecycle-hooks.ts:647` (`hookFunctionsSave`), `:941` (`getLifecycleHooksForScalingMain`).
- W2: pre-fix `deleteInFlightExecution` records no terminal state: pruning on -> only backdated `deletedAt` (`execution-persistence.ts:643`); pruning off -> hard delete.
- W3: #38552 (open, unmerged at check; head `3056e6e82b`; PR body "Fixes #38551") adds `status`/`finished`/`stoppedAt` to the deletion target, a guarded canceled-status update + deletion-only fallback, plus regression tests (zero-output success, cancellation race).
- W4: issue state at post time: open, 1 comment (n8n-assistant bot), Linear GHC-9460.
- W5: author-reported affected versions: 2.32.5 and 2.39.4.

Warranty: a miss here gets a public correction on the comment.

## Comment (as posted)
> @arried — resolution trail for this one, checked today at master@8bff5da: an open fix exists, and master still carries the old shape.
>
> - The gap: with `saveDataSuccessExecution: none`, a finished production execution skips the save path in `hookFunctionsSave` and is handed to `ExecutionPersistence.deleteInFlightExecution()` instead (`execution-lifecycle-hooks.ts:647`; `:941` in scaling mode). That deletion records no terminal state: with pruning enabled it persists only a backdated `deletedAt` (`execution-persistence.ts:643`). Per the fix's description, that is how the zero-output case here could stay stored as `running` indefinitely.
> - The fix: #38552, "finalize unsaved executions before deletion" (@mayanksingh-27) — passes `status` / `finished` / `stoppedAt` into the deletion path in both regular and scaling modes, adds a guarded update so an already-canceled status is preserved, and includes regression coverage for the zero-output case. Open as of this check, so no released tag contains it yet.
> - What that means: keep `saveDataSuccessExecution: "all"` (your A/B/A finding) until the fix ships; #38552 is the thread to watch.
>
> (Checked today at 8bff5da. Verification is the trade; the door is in my profile.)
