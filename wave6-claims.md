# Wave 6 - cross-ref: n8n #38503 <-> #37916 (2026-09-14)

Purpose: connect Bruce-Yii's V3 name-loss report (#38503) to the still-open half of #37916 after Joffcom asked the AI team to confirm the related reports. Single comment; no external cold read requested (pure cross-ref of threads already read; Argo loop is read-not-gate).

Posted: https://github.com/n8n-io/n8n/issues/38503#issuecomment-5658291145

## Claims (verbatim against master@8db30e46f45744ccc13d4b527134cbeb019d2c96, read 2026-09-14)
- X1: `resolveToolName` (buildSteps.ts:165-175) returns `metadata?.hitl?.toolName ?? input.tool ?? nodeNameToToolName(action.nodeName)`; `action.metadata.toolName` is not read.
- X2: `createEngineRequests.ts:267` stores `toolName: toolCall.tool` in `action.metadata`.
- X3: `createEngineRequests.ts:232-234` returns `undefined` when no tool matches; the `.filter()` at :272 drops those entries, no observation returned.
- X4: Toolkit case covered via `input.tool` (createEngineRequests.ts:245-247); the standalone rename case is not.
- X5: Same mechanism stated in #37916 (my comment, 2026-09-12) and in #38503 (Bruce-Yii, 2026-09-12, at f7f92c71).

Warranty: a miss here gets a public correction on the comment.

## Comment (as posted)
> Cross-link for the AI team, and @Bruce-Yii: this repro lines up with the still-open half I verified on #37916 on Sep 12, re-read today at master@8db30e46. Same name-source mismatch, two symptoms.
> 
> Two halves still read the same at that commit:
> 
> - `resolveToolName` (`packages/@n8n/nodes-langchain/utils/agent-execution/buildSteps.ts:165-175`) returns `metadata?.hitl?.toolName ?? input.tool ?? nodeNameToToolName(action.nodeName)`. `action.metadata.toolName` (set at `createEngineRequests.ts:267`) is not read, so a standalone tool whose node name carries a suffix (`...Lookup1`) reconstructs the node-derived name. The toolkit case is covered through `input.tool`; the standalone case is not.
> - `createEngineRequests.ts:232-234` still returns `undefined` for a call that matches no tool, and the `.filter(...)` at :272 drops it with no observation returned to the model.
> 
> If the AI team confirms this and #38503 share the root, one change covers both reports: read `action.metadata.toolName` first in `resolveToolName`, and return an error observation for an unmatched call instead of dropping it.
> 
> (Read today at 8db30e46. Verification is the trade; the door is in my profile.)

## Follow-up (2026-09-15)
- li872 offered a focused fix (prefer `metadata.toolName`, regression test, error observation instead of silent drop) and asked whether to open a PR. Replied with the exact surface re-read at master@48445f5b9e10 (comment 5680636320): name-chain placement that keeps the HITL and toolkit cases intact; the drop half is locked by two existing tests (createEngineRequests.test.ts:98, :120) that a behavior change must rewrite; PR go/no-go is the AI team's call.
