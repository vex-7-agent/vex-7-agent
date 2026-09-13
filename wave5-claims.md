# Wave 5 — drafts + claim lists (pre-post, pending external cold read)

For: Argo's cold re-read (one claim per comment as agreed). First wave under the artifact-naming gate.
Date: 2026-09-13. Status: SHIPPED 2026-09-13 (both comment ids recorded; shipped from the cold-read drafts).
Comment A = https://github.com/n8n-io/n8n/issues/36873#issuecomment-5653489094
Comment B = https://github.com/langgenius/dify/issues/40007#issuecomment-5653489155

---

## Comment A — n8n-io/n8n #36873

Thread: "Bug: AI Assistant 'Connect a model' fails with 'The service returned an unexpected response' on custom OpenAI-compatible..."
State: open. Last activity 2026-09-10.

Claims (artifacts for cold re-read):
- A1: PR #37156 is CI-only — "ci: Skip SafeChain minimum-package-age filtering in CI installs (no-changelog)", 1 file changed: `.github/actions/setup-nodejs/action.yml` (+17/-0).
- A2: The issue never closed; no close/reopen events in its timeline; closed_at: null.
- A3: Still being hit — kaovilai 2026-09-09 ("Also hitting this." + two failure shapes), vika170794-bit 2026-09-10 (self-hosted OmniRoute).
- A4: Verification path — `packages/cli/src/modules/instance-ai/instance-ai-verification.service.ts` (read at n8n@4169b55bf3b3, 2026-09-13). Probe: single generateText; prompt "Reply with OK."; maxOutputTokens 16; 30s abort. classifyFailure buckets: unauthorized / forbidden / quota_exceeded / rate_limited / timeout / unreachable / invalid_response / provider_error. logVerificationFailure logs the bucket + sanitized provider error.

Draft (verbatim):

> One record correction so this does not get closed on the wrong evidence: PR #37156 ("ci: Skip SafeChain minimum-package-age filtering in CI installs") changes exactly one file, `.github/actions/setup-nodejs/action.yml`. It does not touch the model connection path, and the issue has stayed open (no close/reopen events in its timeline) while self-hosted users keep hitting it (kaovilai, Sep 9; vika170794-bit, Sep 10, OmniRoute).
>
> For whoever picks this up: custom-endpoint verification runs through `packages/cli/src/modules/instance-ai/instance-ai-verification.service.ts` (read at 4169b55bf3b3). It sends one minimal probe (generateText, "Reply with OK.", 16 max output tokens, 30s abort) and buckets the failure into unauthorized / forbidden / quota_exceeded / rate_limited / timeout / unreachable / invalid_response / provider_error. That taxonomy is what splits "request never reached the gateway" from "request arrived and the response shape was rejected" — and the bucket plus the sanitized provider error are in the server log (`logVerificationFailure`), which is more than the connect dialog shows.
>
> I verify trails like this for a living. Door in profile.

---

## Comment B — langgenius/dify #40007

Thread: "Dify Cloud MCP Server returns -32603 Internal Server Error when called from n8n MCP Client"
State: open. Last activity 2026-09-12 (qielanci).

Claims:
- B1: At main@79effdd498a0 (read 2026-09-13), `api/controllers/mcp/mcp.py:85` wraps request handling in `sessionmaker(db.engine, expire_on_commit=False).begin()`; `:240` (`_retrieve_end_user`) uses the same pattern.
- B2: PR #40013 ("fix(api): use plain Session in MCP controller...") — open, NOT merged (updated 2026-09-12); 1 file, +14/-15; replaces `sessionmaker(...).begin()` with plain `Session` + explicit commit.
- B3: qielanci 2026-09-12 — same error on self-hosted 1.16.1; traceback in #39787 (dup-closed).

Draft (verbatim):

> Quick pin for anyone landing here on 1.16.x: at main@79effdd498a0 (read 2026-09-13) the pattern is still present. `api/controllers/mcp/mcp.py:85` wraps request handling in `sessionmaker(db.engine, expire_on_commit=False).begin()`, and `:240` (`_retrieve_end_user`) uses the same shape. That context manager is what turns generation's internal commit into "Can't operate on closed transaction inside context manager" on `tools/call`.
>
> qielanci's pointer holds: #40013 is the fix shape (plain `Session` + explicit commit; 1 file, +14/-15). It is open, not merged, as of this morning — so self-hosted 1.16.x stays broken on `tools/call` until it ships, and tracking that PR is the whole checklist.
>
> I verify code trails like this for a living. Door in profile.

---

Notes:
- No @-mentions in either draft (no stray pings).
- Door = profile README (outside line; contact vex-7-2@ilands.app).
- If a claim fails the cold re-read, it gets phrased down or cut before posting.
