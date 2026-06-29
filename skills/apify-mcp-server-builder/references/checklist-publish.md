# Publish Checklist — MCP Server Actor

Pre-publish + post-deploy verification for an MCP server Actor on the Apify Store. Run this checklist every time before `apify push` to production, and again after deployment.

This file complements (does not replace) your README/Store-content checklist, your full revenue/billing audit framework, and your graceful-exit checklist.

This checklist focuses on the **MCP-server-specific** items.

## Pre-publish — code review

### Standby configuration

- [ ] `.actor/actor.json` has `usesStandbyMode: true`
- [ ] `.actor/actor.json` has `webServerMcpPath` set to the actual transport path (`/mcp` for Streamable HTTP, `/sse` for Legacy SSE)
- [ ] `.actor/actor.json` has `webServerIdleTimeoutSecs` explicitly set (default to 300 unless usage data says otherwise)
- [ ] `.actor/actor.json` has `minMemoryMbytes` and `maxMemoryMbytes` set tight (256–512 for Node, 512–1024 for Python with heavy deps)
- [ ] `.actor/actor.json` has `defaultRunOptions.timeoutSecs: 0` (no run timeout — Standby idle owns the lifecycle)

### Server bootstrap

- [ ] `Actor.init()` called once at the top of `main.ts` / `main.py`
- [ ] No `Actor.main()` wrapper around the server (would terminate immediately)
- [ ] No `Actor.exit()` in the success path (Standby kills the container, not your code)
- [ ] `SIGTERM` handler calls `Actor.exit()` for graceful shutdown
- [ ] Server reads `process.env.ACTOR_WEB_SERVER_PORT` (Node) / `os.environ['ACTOR_WEB_SERVER_PORT']` (Python), not a hardcoded port
- [ ] Server binds to `0.0.0.0`, not `localhost` (otherwise unreachable from the Apify edge)

### MCP server

- [ ] Server name and version set (visible to clients via MCP `initialize`)
- [ ] Every tool has a `name`, a `description`, and a complete `inputSchema` with field-level descriptions
- [ ] Every tool's handler validates arguments before doing work (Zod / Pydantic / manual checks)
- [ ] Every tool returns `{ content: [{ type: 'text', text: ... }] }` on success
- [ ] Every error path returns `{ content: [...], isError: true }`
- [ ] No exceptions bubble up uncaught from tool handlers

### PPE charging

- [ ] `.actor/pay_per_event.json` declares every event your code charges
- [ ] Every `Actor.charge()` call is after the successful work completes and before the response is returned
- [ ] No `Actor.charge()` inside catch blocks or error paths
- [ ] No `Actor.charge()` for `apify-actor-start` or `apify-default-dataset-item` (synthetic — platform-managed)
- [ ] Every `Actor.charge()` result is checked for `eventChargeLimitReached`; the response branch clearly communicates the cap-hit to the user
- [ ] Free tools (e.g. `list_resources`, `describe_tool`) are explicitly excluded from charging

### Secrets

- [ ] No API tokens / secret keys in `.actor/actor.json`, `package.json`, or any committed source file
- [ ] Secret env vars set in the Apify Console UI (Actor settings → Environment variables → secret)
- [ ] `.gitignore` and `.dockerignore` exclude `.env`, `secrets/`, `credentials.json`
- [ ] If users supply their own upstream credentials, the README documents the mechanism (query param / header / tool arg) and warns about logging

### Dependencies

- [ ] `@modelcontextprotocol/sdk` (Node) or `mcp` (Python) is at the latest stable
- [ ] `apify` SDK at latest stable
- [ ] No unused heavy dependencies (Playwright, Puppeteer, ML libs) — they multiply cold-start time
- [ ] Lazy imports for any optional heavy library (see `path-c-fromscratch.md` § "Cold start mitigation")

### Source-code confidentiality

- [ ] Internal-only files (session journals, audit/strategy docs, scratch notes) are `.gitignored` (or kept outside the repo). ⚠️ `.gitignore` does NOT exclude **already-tracked** files from `apify push` — those upload anyway.
- [ ] **Bundled TS server (tsup/tsc → `dist/`): upload the BUNDLE only — never `src/` TS, `tests/`, or `*.map`** (source maps reconstruct the TS). Deploy from a **staging dir** with a deploy allowlist (a pre-push script that copies only the allowlisted files into the staging dir before `apify push`). *(Python servers run from `src/` directly, so the source is inherently uploaded/exposed — minify, or keep the server private/off-Store, if the logic is sensitive.)*
- [ ] After `apify push`, list `versions[0].sourceFiles[].name` — ⚠️ **`jq` is often absent**, so `… | jq … || echo ok` silently false-passes; extract names with Python. A bundled server must show ONLY `dist/main.js` (+ README, CHANGELOG, `package*.json`, `.actor/*`, Dockerfile) — **NEVER `src/*.ts`, `tests/`, `*.map`**
- [ ] Set `isSourceCodeHidden: True` — it hides the Console UI ONLY, not the public API (`apify actors info --json` still exposes `sourceFiles[]` content to authenticated users), which is why the upload itself must be clean
- [ ] Verify in incognito browser that the public Actor page has no "Source code" tab; remember already-pushed builds keep their exposed source

## Pre-publish — local test

```bash
APIFY_META_ORIGIN="STANDBY" ACTOR_WEB_SERVER_PORT=8080 apify run -p
```

- [ ] Server starts without errors
- [ ] `Actor.init()` log message appears
- [ ] `npx @modelcontextprotocol/inspector` connects to `http://localhost:8080/mcp`
- [ ] `tools/list` returns the expected tools with correct descriptions and schemas
- [ ] Each tool can be invoked from Inspector and returns the expected response
- [ ] Each tool's error path (invalid args, upstream API down) returns a clean MCP error
- [ ] Server logs show `Charged event ...` lines on successful tool calls (local charges are logged, not billed)
- [ ] Server logs show NO charge on error paths or `isError: true` responses

## Deploy

```bash
apify login
apify push
```

- [ ] Build completes without errors
- [ ] Build log mentions the correct base image
- [ ] No warnings about missing files in the build

## Post-deploy — verification

### Apify Console

- [ ] **Settings → Standby** tab visible and shows status "Ready"
- [ ] Standby URL displayed matches the pattern `https://<username>--<actor-name>.apify.actor/`
- [ ] **Monetization** tab shows your PPE events with correct prices
- [ ] **Source code** tab is hidden (you've set `isSourceCodeHidden: True`)

### Smoke test with curl

```bash
curl -i \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  https://<username>--<actor-name>.apify.actor/mcp
```

- [ ] Returns HTTP 200
- [ ] Response body is JSON-RPC with `result.tools` listing all expected tools

### Full test with MCP Inspector

- [ ] Connect Inspector to the production URL with `Authorization: Bearer $APIFY_API_TOKEN`
- [ ] `tools/list` succeeds and matches local
- [ ] Each tool can be invoked successfully
- [ ] Cold-start delay is ≤ 15 s for the first request after idle
- [ ] Subsequent requests (within `webServerIdleTimeoutSecs`) return in < 1 s

### Billing verification

- [ ] In **Runs** tab, the Standby request appears as a run with the expected events charged
- [ ] Per-event price matches `pay_per_event.json`
- [ ] `apify-actor-start` event fires exactly once per cold start
- [ ] No surprise events charged (this catches code that accidentally charges synthetic events)

### MCP client integration test

Pick a real MCP client (Claude Desktop, Claude Code, Cursor, ChatGPT MCP). Configure it with:

```json
{
  "mcpServers": {
    "my-mcp": {
      "url": "https://<username>--<actor-name>.apify.actor/mcp",
      "headers": { "Authorization": "Bearer $APIFY_API_TOKEN" }
    }
  }
}
```

- [ ] Client lists your tools in its UI
- [ ] Client can invoke each tool and the AI assistant gets correct results
- [ ] Asking the assistant to call your tool 5x in a row works (no session breakage)
- [ ] Charges accumulate correctly in **Billing → Insights**

## After publishing to the Store

- [ ] README has been audited against your Apify Store README/content doctrine
- [ ] Pricing has been audited against your Apify monetization/audit framework
- [ ] Discounts (Bronze/Silver/Gold) opt-in is enabled in the Monetization wizard
- [ ] You've tested via Apify's official MCP client tester Actor (search the [Apify Store](https://apify.com/store) for "tester mcp client")

## After 7 days in the Store

- [ ] **Insights → Run success rate** ≥ 95% (below = code bugs, not infra)
- [ ] **Insights → Cold-start frequency** acceptable for your audience (tune `webServerIdleTimeoutSecs` if not)
- [ ] **Insights → Idle compute %** ≤ 20% of revenue (if higher, revisit the Standby-specific cost concerns in your monetization doctrine)
- [ ] No unaddressed user reports in **Issues** tab
- [ ] First reviews appearing in **Reviews** tab — respond to constructive feedback within 48h

## Iron rules — do not skip

If any of these fail, **delete the deployment** and fix before re-publishing:

- [ ] No secrets visible in Apify Source code tab (incognito browser check)
- [ ] No internal/confidential docs in source files
- [ ] No `Actor.charge()` calls in error paths
- [ ] `eventChargeLimitReached` handled in every tool
- [ ] `usesStandbyMode: true` is set (without it, the Actor is not an MCP server, it's a one-shot Actor that pretends to be one)
