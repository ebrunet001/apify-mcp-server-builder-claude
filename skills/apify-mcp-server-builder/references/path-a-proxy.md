# Path A — Proxy an Existing MCP Server

Use Path A when an MCP server already exists (yours, open-source, or third-party) and you only need Apify to **host it + bill for tool calls**. This is the fastest path to a monetized Apify MCP server when the tool logic is already implemented elsewhere.

This file assumes you've read the main `SKILL.md`. It expands Path A specifics.

## When Path A fits

- ✅ You found a community MCP server (e.g. `@modelcontextprotocol/server-everything`, `mcp-server-filesystem`, vendor-published MCP servers) and want to host + monetize it.
- ✅ You have your own MCP server running stdio or remote HTTP that you want to publish on the Apify Store.
- ✅ You want to combine multiple existing MCP servers behind one endpoint (rare; usually handled with a custom Path C implementation).
- ❌ You need tool-call-level transformations of the upstream server's responses (use Path C instead — proxying makes it harder to insert logic per tool).
- ❌ The upstream MCP server is unreliable and you want to add retries/caching (Path C lets you wrap each tool individually).

## Setup

```bash
apify create my-mcp-server --template ts-mcp-proxy
cd my-mcp-server
```

The template scaffolds:

```
my-mcp-server/
├── .actor/
│   ├── actor.json           # Standby + webServerMcpPath pre-configured
│   ├── pay_per_event.json   # Pre-configured with tool-request event
│   └── Dockerfile
├── src/
│   └── main.ts              # Proxy bootstrap — edit MCP_COMMAND here
├── package.json
└── README.md
```

## Configuring the proxy command

Edit `src/main.ts`:

### Wrap a stdio MCP server

```typescript
const MCP_COMMAND = [
  'npx',
  '@modelcontextprotocol/server-everything',
];
```

Apify launches the npx process inside the container, reads its stdio, and converts to HTTP. The container needs Node + npm (the `apify/actor-node:20` base image has both).

For Python stdio servers:

```typescript
const MCP_COMMAND = [
  'python', '-m', 'my_stdio_mcp_module',
];
```

You'll need a base image that has Python (or install Python in the Dockerfile). Since the proxy itself is Node, this dual-runtime setup is heavier than Path C in Python — consider Path C unless you specifically need to wrap an existing Python stdio MCP server.

### Wrap a remote HTTP MCP server

```typescript
const MCP_COMMAND = [
  'npx', 'mcp-remote',
  'https://upstream-mcp.example.com/mcp',
  '--header',
  'Authorization: Bearer UPSTREAM_API_TOKEN',
];
```

The `mcp-remote` npm package is a stdio-to-HTTP adapter — it lets your Apify proxy speak stdio internally while connecting to a remote HTTPS MCP server.

**Note on the upstream token:** never hardcode it in `actor.json` or `main.ts`. Set it as an environment variable in the Apify Console (Actor settings → Environment variables → secret). Then reference it:

```typescript
const upstreamToken = process.env.UPSTREAM_API_TOKEN ?? '';
const MCP_COMMAND = [
  'npx', 'mcp-remote',
  'https://upstream-mcp.example.com/mcp',
  '--header',
  `Authorization: Bearer ${upstreamToken}`,
];
```

For tokens that should come from the *end user* (each Apify user passes their own upstream credentials), use Actor input fields — but be aware that Standby Actors don't have batch input. You can read input via query parameters or the MCP `arguments` of each tool call. See the "User-supplied upstream credentials" section below.

## Adding charging

The template's default `pay_per_event.json` has one event `tool-request` priced at $0.05. Adjust the price and event taxonomy per your Apify monetization/PPE doctrine.

The proxy already wires `Actor.charge({ eventName: 'tool-request' })` per incoming tool call. To customize:

### Per-tool pricing (Pattern B)

Edit `src/main.ts` to inspect the incoming tool name before charging:

```typescript
// Inside the proxy's tool-call middleware
const toolName = parsedRequest.params.name;
const eventName = toolName.startsWith('expensive_') ? 'expensive-call' : 'cheap-call';
const charge = await Actor.charge({ eventName });
if (charge.eventChargeLimitReached) {
  return mcpError('Cost cap reached.');
}
```

Declare both events in `.actor/pay_per_event.json`:

```json
{
  "cheap-call": {
    "eventTitle": "Cheap tool call",
    "eventDescription": "Per call to read-only / lookup tools.",
    "eventPriceUsd": 0.005
  },
  "expensive-call": {
    "eventTitle": "Expensive tool call",
    "eventDescription": "Per call to LLM-backed or heavy tools.",
    "eventPriceUsd": 0.05
  }
}
```

### Free tools (Pattern C: per-result inside tools)

If you want some tools to be free (e.g. `list_resources`) and others to bill:

```typescript
const FREE_TOOLS = new Set(['list_resources', 'describe_tool']);
if (!FREE_TOOLS.has(toolName)) {
  await Actor.charge({ eventName: 'tool-request' });
}
```

## User-supplied upstream credentials

Pattern: end users want to provide their own OpenAI key, GitHub token, etc. The Standby Actor cannot read these from batch input (because there is no batch input). Options:

### Option 1: Query parameter on the URL

```
https://<user>--<actor>.apify.actor/mcp?upstream_token=sk-...
```

In the proxy, read `req.query.upstream_token` and pass it as a header to the upstream server. Pros: simple. Cons: tokens may end up in logs, browser history.

### Option 2: Custom HTTP header

Have the MCP client send `X-Upstream-Token: sk-...` alongside the Apify `Authorization` header. Read in the proxy.

### Option 3: Tool argument

Add a `credentials` argument to each tool and require the client to send it on every call. Cleanest for security; ugliest for client UX (tokens repeated on every call).

### Option 4: Two-step auth (out-of-band)

The Actor exposes a setup tool (`set_credentials`) that stashes the user's credentials in their session (KV Store keyed by Apify user ID derived from the API token). Subsequent calls look up credentials from the session. Most secure; requires session state and a stateful transport — see `transports.md`.

In all cases, log nothing about the user's upstream credentials and redact them from any error messages.

## When Path A starts hurting

Path A is the fastest path to deploy, but it has limits:

- **Per-tool error handling is harder.** The proxy is generic; inserting "retry this specific tool 3 times if it returns 500" requires patching the proxy middleware, not a clean per-tool function.
- **Per-tool charging gets verbose.** Pattern B with 10+ tools means a long switch/lookup table in the proxy.
- **Adding caching is awkward.** You'd cache at the proxy layer, which can't introspect tool semantics ("this tool's result is safe to cache for 1 hour, that one isn't").
- **Tool schemas come from the upstream server.** If the upstream's schema is wrong/incomplete, you can't fix it without forking.

When you hit two or more of these, migrate to Path C. Keep the upstream server as a library (npm dep, pip dep) and wrap each tool individually with the charging + caching layer you actually want.

## Verification after `apify push`

Same as Path C — see `checklist-publish.md`. Specific to proxies:

1. Confirm the child process starts. Check logs for the `MCP_COMMAND` you set; if it crashes (missing npm package, missing env var), the proxy will keep restarting it.
2. Run `tools/list` via MCP Inspector. The tools should be **exactly** what the upstream MCP server exposes. If you see fewer, something in the proxy is filtering; if you see more, you're connected to the wrong upstream.
3. Invoke each tool and verify the response is identical to running the upstream MCP server locally.
4. Trigger an error in the upstream (invalid args, missing upstream service). The proxy should pass it through as an MCP error response; verify no charge is logged.
