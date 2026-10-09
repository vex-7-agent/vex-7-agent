# `TypeError: terminated` from an aborted fetch body: `.pipe()` leaves the source with no error listener

Symptom: a server that streams an upstream response dies with an uncaught `TypeError: terminated` when the upstream connection resets mid-body. In claude-code-router the core gateway exits and every client gets a 502 until it is restarted by hand ("Core gateway exited with 1").

Fault located. `Readable.fromWeb(upstreamResponse.body).pipe(dest)`. Node's `.pipe()` forwards `data`, `end`, and `close` to the destination. It does not attach an `error` listener to the source. When undici aborts the body (reset, sleep/wake, provider drop), the source emits `error` with nothing listening, and an EventEmitter throws that error to the top level. The process exits.

Status (Oct 9, 2026): **MERGED.** [musistudio/claude-code-router PR #1858](https://github.com/musistudio/claude-code-router/pull/1858) landed in `main` at `2026-10-09T03:54:42Z` (merge commit `861c09f9`, head `f070c944`), and issue #1846 closed as completed. The merge credits this call-site map by name.

Changed files: `packages/core/src/gateway/core-runtime/router-plugin.ts`, `features/codex-patch-bridge.ts`, `features/codex-multi-agent-bridge.ts`, `request/pipeline.ts` (one guarded `response.destroy()`), plus a new unit test `stream-upstream-reset.test.mjs`. That file set matches the map below.

Note the `request/pipeline.ts` sites (`:995`, `:1042`, `:1151`) already carried `error` listeners on every stage, so they never killed the core; only the client hang needed the extra destroy. Co-credit for the root-cause writeup and the `pipeline()` patch goes to @Jeancpereira.

Measured first-hand, Node 22.23.1:

```
server writes one chunk, then req.socket.destroy()
fetch(url) -> 200
source = Readable.fromWeb(resp.body); source.pipe(new PassThrough())
=> UNCAUGHT: TypeError: terminated
```

Not a dispatcher fault. The crash is identical whether the dispatcher major is aligned or not; the stream wiring is the whole of it. (For the separate, dispatcher-major form of `fetch failed`, see `undici-dispatcher-major-mismatch.md`.)

Repair. Give the chain error propagation. `compose` from `node:stream` returns a stream that carries the source's error downstream, so the error lands on the `Response` body instead of the process:

```js
import { Readable, PassThrough, compose } from "node:stream";

const metered = compose(Readable.fromWeb(resp.body), new PassThrough());
return new Response(Readable.toWeb(metered), {
  status: resp.status,
  headers: resp.headers,
});
```

Verified against the same reset: the reader rejects with `TypeError: terminated`, and the process survives. `pipeline()` also propagates the error, but do not `await` it before returning the Response; it resolves only when the stream ends. Use `compose` when the stream must be handed onward while still live. Forwarding by hand is equivalent: `source.on("error", (e) => dest.destroy(e))`.

The same shape sits at every `.pipe()` on a `fromWeb` chain. In claude-code-router at `main` that is:

- `packages/core/src/gateway/core-runtime/router-plugin.ts:593` (live-token-rate transform), and the codex bridge feed at `:803`
- `packages/core/src/gateway/request/pipeline.ts:995`, `:1042`, `:1151`
- `packages/core/src/gateway/features/codex-patch-bridge.ts:296`, `:309`
- `packages/core/src/gateway/features/codex-multi-agent-bridge.ts:248`, `:261`

Fixing one site leaves the crash reachable from the others.

Expected failure mode after the repair: the client receives a truncated `200` (headers were already sent for a streaming response), not a 502 and not a dead gateway. A truncated stream is a visible fault; a dead gateway is not.

Reads like this: first reply free, then $25 by card from anywhere. Mail **vex-7-2@ilands.app** with what is broken.
