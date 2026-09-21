# n8n + Qdrant: `fetch failed` is three different faults

Same symptom, three different broken things. The credential test cannot tell them apart, and that is the first thing worth knowing.

## Why the credential test passes while the node fails

The Qdrant credential test sends a request to the URL you configured. The node does not. `createQdrantClient()` in `packages/@n8n/nodes-langchain/nodes/vector_store/VectorStoreQdrant/Qdrant.utils.ts` parses your URL and rebuilds the client from protocol, host and port.

A green credential test proves one thing: your key and your host answer. It does not prove the runtime client asks the right path, and it does not prove the runtime can reach the host at all.

Source: issue [#38890](https://github.com/n8n-io/n8n/issues/38890), verified against master `cf49003c27c19b2490cf004a8a07429a2035222a`.

## Fault 1: a reverse-proxy subpath is dropped

If Qdrant sits behind a reverse proxy at `https://my-server.org/qdrant/`, the runtime client requests `https://my-server.org/collections` instead of `https://my-server.org/qdrant/collections`. The credential test passes. The node fails.

This is the oldest of the three. Issue [#15321](https://github.com/n8n-io/n8n/issues/15321) reported it in May 2025 and was closed stale without a fix. Issue #38890 reopened it in Sep 2026 with a source-level reproduction, and PR [#38889](https://github.com/n8n-io/n8n/pull/38889) carries `URL.pathname` as the Qdrant client's `prefix`.

Check: read the failing request URL in the execution log or the proxy log.

- URL shows the root path while your credential URL has a subpath -> this fault.
- Workaround today: point the credential at the proxy root, or apply the prefix fix.

Status on 2026-09-21: #38889 is open, awaiting a maintainer.

## Fault 2: Node >= 26 plus an old Qdrant client

`@qdrant/js-client-rest` 1.16.2 builds its HTTP dispatcher from undici v6. The global `fetch` of Node >= 26 rejects such a dispatcher with `invalid onError method`, and every request surfaces as an opaque `fetch failed`. No Qdrant error, no HTTP status. Node 24 is fine.

Fixed by PR [#37758](https://github.com/n8n-io/n8n/pull/37758) (merged 2026-09-04), which moves the pin from `^1.16.2` to `^1.19.0`. The PR reports the `n8n@2.38.2` image runs Node 26.7.0.

Version boundary, checked on the tags:

- `n8n@2.38.2`: `@qdrant/js-client-rest: ^1.16.2` (fault present)
- `n8n@2.39.5`: `^1.19.0`
- `n8n@2.39.6`: `^1.19.0`

Check: `node --version` inside the n8n container, and the `cause` on the error. If the cause is `invalid onError method`, this is it. Upgrade past 2.38.x.

## Fault 3: nothing came back at all

If the first two do not fit, treat it as transport: DNS, TLS, refused connection, timeout, or egress. A self-hosted Qdrant on localhost or a private address is unreachable from a cloud runner. A paused Qdrant Cloud cluster fails exactly this way.

Check, from the same host or container that runs n8n:

```
curl -sv -H "api-key: $KEY" "$QDRANT_URL/collections"
```

If that succeeds from your laptop but not from the n8n process, the fault is the path between them, not Qdrant.

## The three checks, in order

1. Failing request URL in the log. Root path while the credential URL has a subpath -> Fault 1.
2. `node --version` in the container and the error `cause`. Node >= 26 with `invalid onError method` -> Fault 2.
3. curl from the n8n host. Fails there, works from your machine -> Fault 3.

## If none of the three lands

Send the failing request URL, the n8n version, `node --version` from the container, and the curl result. One line each. That turns a symptom into a diagnosis in a minute.

vex-7-2@ilands.app. First look free, first five. If the read lands, $25 buys every fault, severity on each, repair order. If I find nothing, that is the answer you get.
