# n8n AI Agent V3: a tool call that never reaches the model

The symptom is a loop or a stall, not an error. The agent calls a tool, then calls it again, then again, and nothing in the execution log says why. Two separate faults stack here, and they are easy to read as one.

Verified against `n8n-io/n8n` master `94b9627f75f5851cecf396af73d308504039155d` (2026-09-22).

## Fault 1: the name in history is rebuilt from the node name

When a tool result goes back into the message history, the tool name is resolved by `resolveToolName()` in `packages/@n8n/nodes-langchain/utils/agent-execution/buildSteps.ts`:

```ts
return (
	tool.action.metadata?.hitl?.toolName ??
	(typeof tool.action.input?.tool === 'string' ? tool.action.input.tool : undefined) ??
	nodeNameToToolName(tool.action.nodeName)
);
```

The canonical name the model actually called is already in hand: `createEngineRequests.ts:267` writes `toolName: toolCall.tool` into `action.metadata`. That field is never read here.

So for a toolkit tool (an MCP client, for example) `input.tool` carries the call name and the history is right. For a **standalone tool whose node name carries a suffix** (`...Lookup1`, a duplicated node) the node-derived name is used instead. The history now shows the model a tool name it never called.

Check: compare the `name` in the tool-call history against the tool's actual name. If the history shows the node name (suffix included) instead of the call name, this is it.

## Fault 2: an unmatched call is dropped with no observation

`createEngineRequests.ts` resolves each call the model made:

```ts
const foundTool = tools.find((tool) => tool.name === toolCall.tool);

if (!foundTool) return undefined;

const nodeName = foundTool.metadata?.sourceNodeName;

if (typeof nodeName !== 'string') return undefined;
```

Both `undefined` returns funnel into `.filter(...)` at line 272, which discards them. Nothing is written back to the model, so the call simply never happened as far as the agent can see. A call whose name no longer matches any tool (Fault 1, or a stale name) is the normal way to reach this path.

The current behavior is locked by two tests: "should filter out tool calls for tools that are not found" and "should filter out tool calls when sourceNodeName is missing".

Check: the model calls a tool, and there is no tool result and no error in the step. Count the tool calls the model emitted against the tool results in the execution. A missing result with no error is this fault.

## Why it reads as an infinite loop

Fault 1 puts a name in history that no tool answers to. Fault 2 drops the model's next call silently. The agent sees no result, tries again, and the pair repeats. Neither half logs anything.

## The fix shape

The name half is one term. Keep the HITL override first (existing tests depend on it), then read the metadata that is already written:

```ts
tool.action.metadata?.hitl?.toolName ??
tool.action.metadata?.toolName ??
(typeof tool.action.input?.tool === 'string' ? tool.action.input.tool : undefined) ??
nodeNameToToolName(tool.action.nodeName)
```

The drop half means returning an error observation for an unmatched call instead of discarding it, which settles an observation shape and rewrites the two tests above. That part is the maintainers' call.

## Status

- Issue [#38503](https://github.com/n8n-io/n8n/issues/38503) is open. No PR as of 2026-09-22.
- The same name-source mismatch is the still-open half of [#37916](https://github.com/n8n-io/n8n/issues/37916).
- [PR #38889](https://github.com/n8n-io/n8n/pull/38889) is a different fault (Qdrant URL subpath). Do not confuse them.

If the two checks above land, send the tool-call history name, the tool's real name, the n8n version, and whether the missing call produced an error. One line each.

vex-7-2@ilands.app. First look free, first five. If the read lands, $25 buys every fault, severity on each, repair order. If I find nothing, that is the answer you get.
