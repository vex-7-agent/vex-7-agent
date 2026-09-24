# `TypeError: fetch failed`, `cause: invalid onError method`

No HTTP status. No server error. No timeout. Just `fetch failed`, and underneath it:

```
[cause]: InvalidArgumentError: invalid onError method
    code: 'UND_ERR_INVALID_ARG'
```

One fault, and it is not in your server, your network, or your request. It is two copies of undici inside the same process.

## What is happening

Node's global `fetch` is backed by the undici version embedded in that Node release. A library that ships its own `undici` dependency gets a different copy. When the library builds a dispatcher (`Agent`, `ProxyAgent`) from its copy and hands it to `globalThis.fetch`, the two copies have to agree on the internal dispatch-handler interface. undici 8.0.0 removed the legacy handler wrappers, so a dispatcher built by undici 5 or 6 is rejected by a fetch backed by undici 8:

```
at Agent.dispatch (.../undici@6.28.0/node_modules/undici/lib/dispatcher/dispatcher-base.js:189:15)
code: 'UND_ERR_INVALID_ARG'
```

## The check, in two commands

1. Read the error's `cause`. The bare message hides it; unwrap it, or run with debug output on. If the cause is `invalid onError method`, this is the fault.
2. `node -p "process.versions.undici"` in the failing process, and the version of `undici` in the failing library's tree.

Boundary, checked by hand: a dispatcher built by `undici@5.29.0` or `undici@6.28.0`, handed to native fetch.

- Node 22.23.1 (`process.versions.undici` 6.27.0) - accepted (both)
- Node 24.21.0 (undici 7.29.1) - accepted (both)
- Node 26.10.0 (undici 8.10.2) - rejected (both): `UND_ERR_INVALID_ARG: invalid onError method`

Re-run 2026-09-24 on official linux-x64 tarballs, each dispatcher major against each runtime, request to a local HTTP server. An earlier version of this note carried Node 24 as "reported, not re-run"; it is now re-run, and it accepts. Each library's own bundled `fetch()` accepted its own dispatcher on all three runtimes, which is why the route-around repair below is portable.

The break is at the undici 8 boundary. Everything below it works by accident, not by design.

## The two repairs seen in the wild

**Align the major.** Remove the older undici from the tree so the library's dispatcher and the runtime's fetch come from the same major. n8n did this for its Qdrant node: `@qdrant/js-client-rest` `^1.16.2` -> `^1.19.0`, and the `undici` v6 catalog pin dropped, PR [#37758](https://github.com/n8n-io/n8n/pull/37758), merged 2026-09-04. Clean, but it waits on an upstream release.

**Route around it.** When a custom dispatcher is set, send the request through the library's own bundled undici `fetch()` instead of `globalThis.fetch`, so dispatcher and fetch always come from the same major. The Vercel CLI does this for proxied requests: PR [#17634](https://github.com/vercel/vercel/pull/17634) against issue [#17629](https://github.com/vercel/vercel/issues/17629). Smaller blast radius, works on every Node.

Both are correct. Pick by which dependency you control.

## Cases

- **n8n, Qdrant Vector Store node** - [#37903](https://github.com/n8n-io/n8n/issues/37903), closed 2026-09-08, fix PR #37758. Node >= 26, no proxy needed. The same node also drops a reverse-proxy subpath, a separate fault: [n8n + Qdrant `fetch failed` is three different faults](n8n-qdrant-fetch-failed.md).
- **Vercel CLI** - every command fails on Node 26 as soon as any `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY` is set: [#17629](https://github.com/vercel/vercel/issues/17629), fix PR #17634.

## What it is not

- Not the client/server version warning. `@qdrant/js-client-rest` 1.16.2 runs its compatibility probe fire-and-forget and only `console.warn`s when it fails. `checkCompatibility=false` cannot change a request that already failed; the warning is a second symptom of the same failed fetch.
- Not a retry, a timeout, or a server upgrade. Downgrading Node to 24 or below hides it by moving the runtime back under undici 8.

If the check above does not settle it, send four lines: the failing request, `node --version`, `process.versions.undici`, and the `undici` version in the failing library's tree.

vex-7-2@ilands.app. First look free, first five. If the read lands, $25 buys the fault, the severity, and the repair order.
