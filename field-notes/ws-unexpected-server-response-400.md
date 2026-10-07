# `Unexpected server response: 400`: an unhandled `ws` handshake rejection that exits the process

**Symptom.** The process dies with an uncaught error whose message is exactly:

```
Unexpected server response: 400
```

Grepping your own code for that string finds nothing. It does not come from your code. It comes from `ws`.

**Where it comes from.** `ws` emits this message when the server answers the WebSocket opening handshake with a non-101 status (`ws/lib/websocket.js`, `abortHandshake` → `process.nextTick(emitErrorAndClose, ...)`). An `'error'` event on an `EventEmitter` with no listener is rethrown, which on Node is fatal. A refused provider socket is therefore an app-level crash unless someone attached an `error` listener.

**Measured, first-hand** (`ws` 8.22.0 / Node 22.23.1, no proxy):

- Server answers the upgrade with `HTTP/1.1 400`. A client built exactly like a socket helper that attaches only `open` and `close` handlers throws an uncaught `Unexpected server response: 400` and exits nonzero.
- The same client with one `socket.on('error', ...)` receives the error, the socket closes (1006), and the process survives.

**The trap in the trace.** The captured stack has ten frames and every one is `ws` or a node internal (`ws/lib/websocket.js`, `node:_http_client`, `node:internal/streams`). There are zero app frames. So the trace names `ws`, not the caller. You cannot find the owner from the stack; the owner is the site that constructed the socket.

**Owner vs helper (OpenClaw).** `openProviderWebSocket` returns the socket to its caller (`src/infra/net/provider-websocket.ts`; ctor at `:167`, and it attaches only `open` at `:181` and `close` at `:182`). The contract, in practice: the caller owns the handshake outcome. The bundled Deepgram caller honors this (`extensions/deepgram/audio-flux.ts:298` attaches an error listener), so the in-repo caller is safe. The crash class bites a third-party plugin that uses the SDK helper, re-exported at `src/plugin-sdk/provider-http.ts:54`, and attaches only `open`/`close`.

**Contained fixes.**

1. At the helper: own the handshake outcome. Attach a default one-shot `error` listener that turns a non-101 into an operation-level failure, or await the handshake and reject on non-101, and let the caller override.
2. At the caller: a single `socket.on('error', ...)` at the creation site. That is the entire containment.

**Do not widen the classifier.** `src/infra/unhandled-rejections.ts:96` exact-matches one pre-handshake `ws` message (`websocket was closed before the connection was established`) and deliberately keeps this one fatal. A dropped provider socket is a real failure, not noise; the fix is to own it where the socket is created.

**Repro.** Two modes: `node repro.js nolistener` (crash) and `node repro.js error` (survives).

```js
const http = require('http');
const WebSocket = require('ws');
const mode = process.argv[2] || 'nolistener';

const server = http.createServer();
server.on('upgrade', (req, socket) => {
  socket.write('HTTP/1.1 400 Bad Request\r\nConnection: close\r\n\r\n');
  socket.destroy();
});
server.listen(0, () => {
  const ws = new WebSocket(`ws://127.0.0.1:${server.address().port}/`);
  ws.on('open', () => console.log('OPEN'));
  ws.on('close', (code) => console.log('CLOSE', code));
  if (mode === 'error') ws.on('error', (e) => console.log('CAUGHT', e.message));
  setTimeout(() => { console.log('SURVIVED'); server.close(); process.exit(0); }, 600);
});
process.on('uncaughtException', (e) => {
  console.log('UNCAUGHT:', e.message);
  const frames = (e.stack || '').split('\n').slice(1).map(l => l.trim());
  console.log('APP_FRAMES:', frames.filter(l => !/node:|node_modules\/ws\/|\/ws\/lib\//.test(l)).length);
  process.exit(7);
});
```

Observed: `nolistener` → `UNCAUGHT: Unexpected server response: 400`, `APP_FRAMES: 0`, exit 7. `error` → `CAUGHT ...`, `CLOSE 1006`, `SURVIVED`, exit 0.

**Pinned.** openclaw/openclaw `main` @ `4c3a5b65050b` (2026-10-07). File shas: `provider-websocket.ts` `7f7ecf0ade`, `provider-http.ts` `c3b7110fe6`, `audio-flux.ts` `b06e16a8d5`, `unhandled-rejections.ts` `b0fe8136ed`. Read them at that commit; line anchors drift.
