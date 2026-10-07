# `Unsupported input item type: item_reference`: vLLM's `/v1/responses` rejects a type its own schema accepts

**Symptom.** An OpenAI-compatible client talks to a vLLM or LiteLLM gateway. The app reports a generic connection failure. On the n8n side the string is `The service returned an unexpected response`; the vLLM server log carries the real reason:

```
[ERROR] Unsupported input item type: item_reference (gemma)
```

The item type is legal OpenAI Responses API. The gateway rejects the whole request anyway.

**Where the string comes from.** Not the client. vLLM's Responses input converter walks the input items and maps each one to a chat message. It handles a fixed set of shapes (reasoning item, output message, function-call output, and plain dicts that carry a `role`) and then falls through:

`vllm/entrypoints/openai/responses/utils.py:362-366`, vllm-project/vllm `main` @ `554340f3d3259e321be4c07282be7a02a5aeef83`:

```python
    item_type = item.get("type") if isinstance(item, dict) else item.type
    raise VLLMValidationError(
        f"Unsupported input item type: {item_type}",
        parameter="input",
    )
```

The call chain is `construct_input_messages` → `construct_chat_messages_with_tool_call` → `_construct_message_from_response_item` (same file). No branch matches `type == "item_reference"`, so it lands on the raise.

**Why the item got past the schema.** `ResponseInputOutputItem` is `ResponseInputItemParam | ResponseOutputItem`, and `ResponseInputItemParam` is the OpenAI SDK union, which *includes* `ItemReference`:

`openai-python` `src/openai/types/responses/response_input_item_param.py:692`:

```python
class ItemReference(TypedDict, total=False):
    """An internal identifier for an item to reference."""
    id: Required[str]
    type: Optional[Literal["item_reference"]]
```

So the request validates. The schema accepts the type; the converter has no handler for it. Client and gateway disagree about the same type name: one treats it as legal input, the other as unsupported.

**What sends it.** `item_reference` is the Responses API's way to point at a prior item by id instead of re-sending it (e.g. `{"type": "item_reference", "id": "msg_..."}`). Clients that keep server-side response state, or are configured to, emit it; openai-node has a dedicated case for the type (`src/lib/responses/ResponseInputItems.ts:126`). A stateless gateway keeps no prior-item store, so even a parser that recognized the reference would have nothing to resolve it against. Against such a gateway the client should not send references at all.

**Two fixes, one per side.**

1. Gateway (vLLM): resolve `item_reference` against `prev_response_output` / `prev_msg` when those are present, or skip-and-log an unresolvable reference rather than failing the entire request. The error already names the offending type, which is the right diagnostic behavior.
2. Client: force the chat-completions route when the gateway does not implement the full Responses API. In `@langchain/openai` the switch is `useResponsesApi` (`libs/providers/langchain-openai/src/chat_models/index.ts`); the Vercel AI SDK exposes the Responses and chat-completions endpoints as separate providers.

**One generic string, more than one fault.** The same n8n report also logs `KeyError: 'role'` on the GLM path. That is a *different* root cause wearing the same UI string. I have not traced it and do not claim a mechanism for it.

**Symptom-to-owner map.**

- `Unsupported input item type: X` → vLLM `responses/utils.py` converter, no branch for `X`. The type list is the converter's, not the schema's.
- n8n `The service returned an unexpected response` → a UI bucket over several distinct failures. The real bucket plus the sanitized provider error is in the server log (`packages/cli/src/modules/instance-ai/instance-ai-verification.service.ts`, `logVerificationFailure`). Read the server log, not the dialog.

**Pinned.** vllm-project/vllm `main` @ `554340f3d3259e321be4c07282be7a02a5aeef83` (2026-10-07), `vllm/entrypoints/openai/responses/utils.py` 423 lines. openai-python `main`, `response_input_item_param.py` 770 lines. Read at those commits; line anchors drift.

---

## Two shapes of the same type, depending on your build

`item_reference` fails differently on an older vLLM than on current `main`, and the two
strings send you to different places.

**Before the guard (e.g. `vllm/vllm-openai:v0.28.0`, built 2026-08-25).** The converter's
tail was a bare `return item`, so the reference dict reached the chat renderer and the
server returned **HTTP 500** with `KeyError: 'role'` at `vllm/entrypoints/chat_utils.py`:

```
{"error":{"message":"'role'","type":"InternalServerError","param":null,"code":500}}
```

**After the guard (`main`, PR #55974 merged 2026-09-09, merge commit `114abd1c1131`).** The
same item is rejected with a named 400:

```
Unsupported input item type: item_reference
```

So the crash is fixed, and the *rejection* is not. `item_reference` is still unusable against
a gateway that keeps no prior-item store. Searching either string should land you here:

- `KeyError: 'role'` on `POST /v1/responses` → pre-#55974 build, unguarded `return item`.
- `Unsupported input item type: X` → post-#55974 build, converter has no branch for `X`.

The severity sweep behind the crash path (3 types returning 500, 11 returning a leaked Python
`TypeError` as a 400, out of the 32 SDK input item types) is on vllm-project/vllm#60396.
