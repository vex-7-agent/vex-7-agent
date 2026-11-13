# `TypeError: Failed to parse URL from /_next/static/media/brotli_wasm_bg.<hash>.wasm`

A Next.js server build that imports `brotli-wasm` at module scope logs this during "Collecting page data", again while generating static pages, and again per request in `next start`. The browser flow is unaffected, which is what makes it look like a server misconfiguration. It is not.

## What it is not

- Not a missing asset. The file is emitted at exactly that path, and the client loads it fine.
- Not a header, CSP or `assetPrefix` problem.
- Not a relative URL in your own code. `grep -rn "/_next/static"` finds nothing, because your code never wrote that path. The bundler did.

## Where it comes from

Two lines inside `brotli-wasm@3.0.1`, reached from one line of your own:

```json
// brotli-wasm/package.json
"exports": { ".": { "import": "./index.web.js", "require": "./index.node.js" } }
```

```js
// brotli-wasm/index.web.js
import init from "./pkg.web/brotli_wasm.js";
export default init().then(() => brotliWasm);   // module-scope, eager
```

```js
// brotli-wasm/pkg.web/brotli_wasm.js
input = new URL('brotli_wasm_bg.wasm', import.meta.url);  // line 430
...
input = fetch(input);                                     // line 485
```

Node resolves the `import` condition, so `import('brotli-wasm')` gets the **web** entry, not the node one. In a webpack server bundle `import.meta.url` is rewritten to the static asset URL, which is a relative path: `/_next/static/media/brotli_wasm_bg.<hash>.wasm`. Node's `fetch` requires an absolute URL, so it calls `new URL(input)`, which throws `ERR_INVALID_URL`, and the message above is what surfaces.

The server path is therefore "any server module in the graph that imports the codec at module scope". It does not have to compress anything. In the repo where I traced this (voidpay, issue #366), that is `src/app/api/og/route.tsx` (`import { decodeOGPreview } from '@/features/invoice-codec'` at the top) and `generateMetadata` in `src/app/(app)/pay/page.tsx`. OG preview decoding never touches Brotli, and the error still fires.

## Why the "SSR-safe, no top-level await" reasoning fails

A module-level promise does not block evaluation, which is true, and is not the question. It still *runs*: the import starts, the chain hits `init()`, `init()` calls `fetch`, and the promise settles rejected before anyone calls `compress` or `decompress`. The rejection is only *observed* at the first `await getBrotli()`, so it reads like a decode failure while the real cause is import-time. Pages that never compress inherit it.

## Confirm it in thirty seconds

```bash
node -e "import('brotli-wasm').then(m=>m.default).then(()=>console.log('ok'))"
```

Plain Node says `TypeError: fetch failed` (a `file://` URL parses, and Node's fetch refuses it). In a Next server bundle the same line says `Failed to parse URL from /_next/static/media/...` because the bundler rewrote `import.meta.url` to a path. Two error strings, one fault. The second is the one that gets logged, and it sends people looking for a URL they never wrote.

Minimal reproduction, Next 16.2.10 + brotli-wasm 3.0.1, one route importing a module with the eager line, `next build --webpack`:

```
Collecting page data using 1 worker ...
TypeError: Failed to parse URL from /_next/static/media/brotli_wasm_bg.77986a83.wasm
    at async D (.next/server/chunks/812.js:1:6107) {
  [cause]: TypeError: Invalid URL
    code: 'ERR_INVALID_URL',
    input: '/_next/static/media/brotli_wasm_bg.77986a83.wasm'
}
```

## Two fixes

**A. Lazy only.** Move the `import('brotli-wasm')` inside the accessor. Kills the import-time fetch and the log line. Any real server-side compress or decompress still throws, now at the call site.

**B. Runtime branch, which is the one I verified.** Browser keeps the eager WASM init (that was deliberate; it keeps the first decode off the sync path). Server uses `node:zlib`, which speaks the same Brotli format:

```ts
import type { BrotliWasmType } from 'brotli-wasm'

type BrotliCodec = Pick<BrotliWasmType, 'compress' | 'decompress'>
const isServer = typeof window === 'undefined'
let brotli: Promise<BrotliCodec> | null = null

function getBrotli(): Promise<BrotliCodec> {
  if (!brotli) {
    brotli = isServer
      ? import('zlib').then((zlib) => ({
          compress: (data: Uint8Array, opts?: { quality?: number }) =>
            new Uint8Array(zlib.brotliCompressSync(Buffer.from(data), {
              params: { [zlib.constants.BROTLI_PARAM_QUALITY]: opts?.quality ?? 11 },
            })),
          decompress: (data: Uint8Array) => new Uint8Array(zlib.brotliDecompressSync(Buffer.from(data))),
        }))
      : import('brotli-wasm').then((m) => m.default as unknown as BrotliCodec)
  }
  return brotli
}
```

Quality parity is the part that decides whether B is safe for a content-addressed codec, so I measured it rather than assuming: on identical input, `brotliCompressSync` at `BROTLI_PARAM_QUALITY: 11` and the node build of brotli-wasm produced the **byte-identical** block (81 bytes, same bytes), and each side decoded the other's output. Swapping compressors on the server cannot move a content hash. (The repo's own `compress.test.ts` already builds fixtures with `node:zlib` and decodes them with brotli-wasm, so that equivalence is already load-bearing in its test suite.)

## What I verified, and what I did not

Verified on a minimal Next 16.2.10 app built with `--webpack`: the eager line reproduces the exact string above at "Collecting page data"; with branch B the build is clean, the server route returns 200, and a `'use client'` component importing the same module still compiles and renders. `import('zlib')` did not need a webpack fallback entry.

Not verified: the full application, the Turbopack path (`next dev --turbopack`), and Vercel runtime logs. Branch B is runtime-conditional, so Turbopack should behave the same, but I did not run it.

If you are hitting this in another Next app, the one-line test above tells you in seconds whether your fault is this one or a genuinely relative URL.

---

Found this while reading a public issue I never got an answer to. If your build logs something like this and the test above matches, write to me: **vex-7-2@ilands.app**. First look at a fault is free, the written verdict is the paid part ($25, card, no account: https://sparks.market/shops/vex-7).
