# Path C — Building an MCP Server Actor from Scratch

The full reference for building an MCP server Actor on Apify when no existing MCP server or OpenAPI spec covers your tools. Covers Node and Python in parallel.

This file assumes you've read the main `SKILL.md`. It expands the skeleton examples and adds production patterns (error envelope, cold start, stateful sessions, lazy loading, testing).

## Project structure

### Node (TypeScript)

```
my-mcp-server/
├── .actor/
│   ├── actor.json
│   ├── pay_per_event.json
│   ├── input_schema.json        # optional — Standby Actors don't need batch input
│   └── Dockerfile
├── src/
│   ├── main.ts                  # Actor.init() + Express + MCP server bootstrap
│   ├── server.ts                # MCP server + tool registration
│   ├── tools/
│   │   ├── index.ts             # tool registry
│   │   └── githubStars.ts       # one tool per file
│   ├── ppe/
│   │   └── charge.ts            # Actor.charge wrapper with charge-cap handling
│   └── utils/
│       └── errors.ts            # MCP error envelope helper
├── tests/
│   └── tools.test.ts
├── package.json
├── tsconfig.json
├── .dockerignore
└── README.md
```

### Python

```
my-mcp-server/
├── .actor/
│   ├── actor.json
│   ├── pay_per_event.json
│   ├── input_schema.json
│   └── Dockerfile
├── src/
│   ├── main.py                  # Actor entry + FastMCP bootstrap
│   ├── server.py                # FastMCP instance + tool registration
│   ├── tools/
│   │   ├── __init__.py
│   │   └── github_stars.py
│   ├── ppe/
│   │   └── charge.py            # Actor.charge wrapper
│   └── utils/
│       └── errors.py
├── tests/
│   └── test_tools.py
├── pyproject.toml               # or requirements.txt
└── README.md
```

## `.actor/actor.json` — full example

```json
{
  "actorSpecification": 1,
  "name": "github-stars-mcp",
  "title": "GitHub Stars MCP Server",
  "version": "0.1",
  "buildTag": "latest",
  "usesStandbyMode": true,
  "webServerMcpPath": "/mcp",
  "webServerIdleTimeoutSecs": 300,
  "minMemoryMbytes": 256,
  "maxMemoryMbytes": 512,
  "defaultRunOptions": {
    "build": "latest",
    "memoryMbytes": 256,
    "timeoutSecs": 0
  },
  "dockerfile": "./Dockerfile",
  "input": "./input_schema.json",
  "storages": {
    "dataset": "./dataset_schema.json"
  },
  "categories": ["MCP_SERVERS", "AI", "INTEGRATIONS"]
}
```

Notes:
- `timeoutSecs: 0` means no timeout — the Standby container shuts down via idle timeout, not run timeout. **Required for MCP.** A non-zero `timeoutSecs` will kill the container mid-session.
- `dataset_schema.json` is optional for MCP servers; some Actors push billing/audit events to the default dataset and benefit from typing it. If you don't, omit `storages`.
- `input_schema.json` is optional but recommended — even if Standby doesn't need batch input, users running the Actor in normal mode (e.g. for debugging) appreciate a clean input form. Keep it minimal: a debug flag, optional auth tokens for third-party APIs.

## `.actor/pay_per_event.json` — full example

The shape is a flat object keyed by event name. Each event declares `eventTitle`, `eventDescription`, and `eventPriceUsd`. Do NOT declare `apify-actor-start` here — it is platform-managed.

```json
{
  "tool-call-github-stars": {
    "eventTitle": "GitHub stars lookup",
    "eventDescription": "Per successful star-count return for a public GitHub repository.",
    "eventPriceUsd": 0.005
  }
}
```

For multi-tool servers, apply the three taxonomy patterns (flat, per-tool tiered, per-result) from your Apify monetization/PPE doctrine.

**Price point for first-time publishers:** $0.001–$0.01 per tool call is the empirical range for read-only lookups; $0.02–$0.10 for AI-LLM-backed tools. Start at the low end during the calibration window, then raise (note that price increases are subject to a delay before they take effect). **Do not guess a price** — work out the pricing strategy before publishing to the Store.

## `.actor/Dockerfile`

### Node

```Dockerfile
FROM apify/actor-node:20

COPY package*.json ./
RUN npm install --omit=dev --omit=optional

COPY . ./

CMD ["npm", "start"]
```

### Python

```Dockerfile
FROM apify/actor-python:3.13

COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt

COPY . ./

CMD ["python", "-m", "src.main"]
```

Both base images include the Apify SDK and standard Node/Python tooling. Don't pull a Playwright base image unless you need browser automation inside your tools.

## `package.json` (Node)

```json
{
  "name": "github-stars-mcp",
  "version": "0.1.0",
  "type": "module",
  "scripts": {
    "start": "node --enable-source-maps dist/main.js",
    "build": "tsc",
    "dev": "APIFY_META_ORIGIN=STANDBY ACTOR_WEB_SERVER_PORT=8080 apify run -p",
    "test": "vitest run"
  },
  "dependencies": {
    "@modelcontextprotocol/sdk": "^1.0.0",
    "apify": "^3.4.0",
    "express": "^4.19.0",
    "zod": "^3.23.0"
  },
  "devDependencies": {
    "@types/express": "^5.0.0",
    "@types/node": "^20.14.0",
    "typescript": "^5.5.0",
    "vitest": "^2.0.0"
  }
}
```

## `pyproject.toml` / `requirements.txt` (Python)

```
apify>=2.0.0
mcp>=1.0.0
httpx>=0.27.0
```

`mcp` is the official Anthropic MCP package; `FastMCP` is the high-level decorator API inside it.

## `src/main.ts` — full Node example

> **API choice:** this reference uses the **low-level `Server` API** from `@modelcontextprotocol/sdk/server/index.js` with explicit `setRequestHandler(CallToolRequestSchema, …)` / `setRequestHandler(ListToolsRequestSchema, …)`. This gives full control over the JSON-RPC envelope, supports middleware patterns (charging wrappers, request logging), and makes multi-tool routing explicit. The main `SKILL.md` skeleton uses the **high-level `McpServer` API** (`mcp.tool(name, desc, schema, handler)`) which wraps the same primitives and is shorter for simple servers. Both are first-party SDK APIs. Don't mix them in the same Actor.


```typescript
import { Actor, log } from 'apify';
import express from 'express';
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { registerTools } from './server.js';

await Actor.init();

const mcp = new Server(
  { name: 'github-stars-mcp', version: '0.1.0' },
  { capabilities: { tools: {} } },
);

registerTools(mcp);

const app = express();
app.use(express.json({ limit: '4mb' }));

// MCP Streamable HTTP endpoint
app.post('/mcp', async (req, res) => {
  try {
    const transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: undefined,  // stateless
    });
    await mcp.connect(transport);
    await transport.handleRequest(req, res, req.body);
  } catch (err) {
    log.exception(err as Error, 'MCP transport error');
    if (!res.headersSent) res.status(500).end();
  }
});

// Health check at GET /
// Apify does NOT require a specific health-probe path — Standby readiness is determined
// by the container binding to ACTOR_WEB_SERVER_PORT successfully. The GET / handler below
// is OPTIONAL: it (a) gives a clean response when users hit the bare Standby URL in a browser,
// and (b) confirms the server is alive without invoking MCP. Strongly recommended; not required.
app.get('/', (_req, res) => {
  res.json({ status: 'ok', server: 'github-stars-mcp' });
});

const port = parseInt(process.env.ACTOR_WEB_SERVER_PORT ?? '3001', 10);
app.listen(port, () => {
  log.info(`MCP server listening on port ${port}`);
});

// Do NOT call Actor.exit() — Standby owns the lifecycle.
// Graceful shutdown on SIGTERM (Apify sends this on idle timeout):
process.on('SIGTERM', async () => {
  log.info('SIGTERM received — shutting down');
  await Actor.exit();
});
```

## `src/server.ts` — tool registration

```typescript
import { Server } from '@modelcontextprotocol/sdk/server/index.js';
import { CallToolRequestSchema, ListToolsRequestSchema } from '@modelcontextprotocol/sdk/types.js';
import { z } from 'zod';
import { getGithubRepoStars } from './tools/githubStars.js';
import { chargeEvent } from './ppe/charge.js';
import { mcpError, mcpText } from './utils/errors.js';

const tools = [
  {
    name: 'get_github_repo_stars',
    description: 'Returns the current star count for a public GitHub repository.',
    inputSchema: {
      type: 'object',
      properties: {
        owner: { type: 'string', description: 'GitHub username or organization' },
        repo: { type: 'string', description: 'Repository name' },
      },
      required: ['owner', 'repo'],
    },
  },
] as const;

export function registerTools(mcp: Server) {
  mcp.setRequestHandler(ListToolsRequestSchema, async () => ({ tools }));

  mcp.setRequestHandler(CallToolRequestSchema, async (req) => {
    const { name, arguments: args } = req.params;

    try {
      if (name === 'get_github_repo_stars') {
        const parsed = z.object({ owner: z.string(), repo: z.string() }).parse(args);
        const result = await getGithubRepoStars(parsed);

        const charged = await chargeEvent({ eventName: 'tool-call-github-stars' });
        if (!charged) {
          return mcpText('Cost cap reached for this session. Result was delivered; increase cap to continue.');
        }
        return mcpText(JSON.stringify(result));
      }
      return mcpError(`Unknown tool: ${name}`);
    } catch (err) {
      return mcpError((err as Error).message);
    }
  });
}
```

## `src/tools/githubStars.ts` — tool implementation

```typescript
export async function getGithubRepoStars(input: { owner: string; repo: string }) {
  const url = `https://api.github.com/repos/${encodeURIComponent(input.owner)}/${encodeURIComponent(input.repo)}`;
  const r = await fetch(url, { headers: { 'User-Agent': 'github-stars-mcp/0.1' } });
  if (r.status === 404) {
    return { found: false, owner: input.owner, repo: input.repo };
  }
  if (!r.ok) {
    throw new Error(`GitHub API ${r.status}: ${await r.text()}`);
  }
  const data = await r.json() as { stargazers_count: number; full_name: string };
  return { found: true, full_name: data.full_name, stars: data.stargazers_count };
}
```

## `src/ppe/charge.ts` — PPE wrapper

```typescript
import { Actor, log } from 'apify';

export async function chargeEvent(opts: { eventName: string; count?: number }): Promise<boolean> {
  const count = opts.count ?? 1;
  if (count <= 0) return false;

  const result = await Actor.charge({ eventName: opts.eventName, count });

  if (result.eventChargeLimitReached) {
    log.info(`Event ${opts.eventName} not charged: limit reached for this user.`);
    return false;
  }

  log.info(`Charged event ${opts.eventName} × ${count}`);
  return true;
}
```

All charging in the Actor goes through this helper. New code can't bypass the rules accidentally. Follow your full PPE implementation doctrine.

## `src/utils/errors.ts` — MCP error envelope

```typescript
import type { CallToolResult } from '@modelcontextprotocol/sdk/types.js';

export function mcpText(text: string): CallToolResult {
  return { content: [{ type: 'text', text }] };
}

export function mcpError(message: string): CallToolResult {
  return { content: [{ type: 'text', text: `Error: ${message}` }], isError: true };
}
```

The `isError: true` flag is part of the MCP spec. Clients use it to distinguish a successful "no results" answer from a failed call. **Never charge on `isError: true`.**

## Python equivalent — `src/main.py`

```python
import asyncio
import os
from apify import Actor
from mcp.server.fastmcp import FastMCP

from .tools.github_stars import get_github_repo_stars
from .ppe.charge import charge_event


async def main() -> None:
    async with Actor:
        mcp = FastMCP('github-stars-mcp')

        @mcp.tool()
        async def get_github_repo_stars_tool(owner: str, repo: str) -> dict:
            """Returns the current star count for a public GitHub repository."""
            try:
                result = await get_github_repo_stars(owner, repo)
            except Exception as e:
                return {'error': str(e), 'isError': True}

            charged = await charge_event('tool-call-github-stars')
            if not charged:
                return {'note': 'Cost cap reached. Result delivered; increase cap to continue.', **result}
            return result

        port = int(os.environ.get('ACTOR_WEB_SERVER_PORT', '3001'))
        Actor.log.info(f'MCP server starting on port {port}')

        await mcp.run_streamable_http_async(
            host='0.0.0.0',
            port=port,
            path='/mcp',
        )


if __name__ == '__main__':
    asyncio.run(main())
```

## Python — `src/ppe/charge.py`

```python
from apify import Actor


async def charge_event(event_name: str, count: int = 1) -> bool:
    if count <= 0:
        return False

    result = await Actor.charge(event_name=event_name, count=count)

    if result.event_charge_limit_reached:
        Actor.log.info(f'Event {event_name} not charged: limit reached.')
        return False

    Actor.log.info(f'Charged event {event_name} x {count}')
    return True
```

## Python — `src/tools/github_stars.py`

```python
import httpx


async def get_github_repo_stars(owner: str, repo: str) -> dict:
    url = f'https://api.github.com/repos/{owner}/{repo}'
    async with httpx.AsyncClient(timeout=10.0) as client:
        r = await client.get(url, headers={'User-Agent': 'github-stars-mcp/0.1'})
        if r.status_code == 404:
            return {'found': False, 'owner': owner, 'repo': repo}
        r.raise_for_status()
        data = r.json()
    return {
        'found': True,
        'full_name': data['full_name'],
        'stars': data['stargazers_count'],
    }
```

## Adding PPE to generator-produced code (Path B output)

An OpenAPI-to-MCP generator Actor produces working MCP servers but does not wire `Actor.charge()`. You must add it after generation. The minimal edit:

1. Open every generated tool file.
2. After the API call succeeds and before returning the response, insert `await chargeEvent({ eventName: '<tool-name>-call' })`.
3. Check the result and branch to a "cap reached" message.

If the generator produces 20 tools, this is repetitive but mechanical. Consider a single shared middleware: wrap the MCP request handler with a `withCharging` decorator that:
- Reads the tool name from the incoming request
- Runs the original handler
- Charges only if the result has no `isError`

This keeps each tool file unchanged from the generator output.

## Cold start mitigation

Cold starts on Standby Actors take 5–15 seconds (Docker pull, Node startup, dependency import). For the AI client, that 15 s is a perceived hang. Mitigations:

### 1. Lazy load heavy dependencies

Bad:

```typescript
import { hugeMlLibrary } from 'huge-ml-library';  // loads at startup
```

Good:

```typescript
async function runHeavyTool(args) {
  const { hugeMlLibrary } = await import('huge-ml-library');  // loads on first use
  return hugeMlLibrary.process(args);
}
```

The first tool call still pays the import cost but only if that tool is actually called.

### 2. Pre-warm caches asynchronously, not synchronously

Don't block `Actor.init()` on cache prefetches. Fire-and-forget:

```typescript
await Actor.init();
prewarmCachesInBackground();  // no await — runs while server is already listening
app.listen(port);
```

### 3. Return progress notifications for known-slow tools

MCP supports progress notifications. If a tool will take >2 s, send a progress update at 500 ms so the client knows the call is alive.

### 4. Do not "keep-alive" by self-pinging

Tempting fix: cron a request to your Standby URL every 4 minutes to keep it warm. **Don't.** Apify bills idle compute. You're paying to avoid cold starts that affect users who aren't there. Either accept the 5–15 s cold start as a UX cost or increase `webServerIdleTimeoutSecs`.

## Stateful vs stateless MCP sessions

The Streamable HTTP transport supports two modes:

| Mode | When | Apify trade-off |
|---|---|---|
| **Stateless** (`sessionIdGenerator: undefined`) | Each request is independent. Recommended default. | Simpler; survives Apify load-balancing across multiple Standby containers. |
| **Stateful** (`sessionIdGenerator: () => randomUUID()`) | The server holds session memory (e.g. running conversation context, large datasets in memory). | Requires session affinity — Apify Standby may rebalance and break in-memory sessions. Use only with external state (KV Store, Redis). |

For the vast majority of MCP servers (per-tool-call lookups), stateless is correct. Stateful is for agent-style MCPs that maintain conversational state.

## Third-party API rate limits and the shared-IP trap

If your tool calls a third-party REST API (GitHub, OpenAI, Stripe, etc.) without per-user credentials, your Actor's egress IP is **shared** across all Apify users invoking your Actor. The third party sees one IP making aggregate calls and may rate-limit or block it.

**GitHub unauthenticated REST API is the canonical example:** 60 requests/hour/IP. A modestly successful Actor blows that limit in seconds.

Three mitigations, in increasing order of effort:

### 1. Cache aggressively in the KV Store

For idempotent reads (repo metadata, public profiles), cache responses in the Apify KV Store with a sensible TTL. Re-uses across users.

```typescript
import { Actor } from 'apify';

async function cachedFetch<T>(key: string, fetcher: () => Promise<T>, ttlMs = 60 * 60 * 1000): Promise<T> {
  const store = await Actor.openKeyValueStore();
  const hit = await store.getValue<{ data: T; ts: number }>(key);
  if (hit && Date.now() - hit.ts < ttlMs) return hit.data;
  const fresh = await fetcher();
  await store.setValue(key, { data: fresh, ts: Date.now() });
  return fresh;
}

const data = await cachedFetch(`github:${owner}:${repo}`, () => fetchGithub(owner, repo));
```

The same KV Store cache pattern lives in your general Apify scraper-patterns reference, with the full helper and idiomatic naming conventions.

### 2. Accept user-supplied credentials per call

If the third party has a higher per-user rate limit (GitHub authenticated REST = 5,000 req/h/user), require the user to pass their token:

```typescript
mcp.tool('get_github_repo_stars', '...', { owner: z.string(), repo: z.string(), github_token: z.string().optional() }, async ({ owner, repo, github_token }) => {
  const headers = { 'User-Agent': 'github-stars-mcp/0.1' };
  if (github_token) headers['Authorization'] = `Bearer ${github_token}`;
  // ...
});
```

Document the trade-off in the README: faster + higher limits with a token, OR throttled shared-pool without one. Never log the token.

### 3. Route via Apify proxy

For tools that scrape (not REST), use `Actor.createProxyConfiguration({ groups: ['RESIDENTIAL'] })` to spread requests across many IPs. Overkill for documented REST APIs; reserve it for the cases where it actually fits.

**Decide before publishing.** Hitting a 429 mid-tool-call returns `isError: true`, which (correctly per the skill's PPE rules) does not charge. But repeated 429s show up as user-visible failures and tank the Actor's success rate.

## Testing strategy

### Unit tests (no MCP)

Each tool is a pure function. Test it directly:

```typescript
import { describe, it, expect } from 'vitest';
import { getGithubRepoStars } from '../src/tools/githubStars';

describe('getGithubRepoStars', () => {
  it('returns stars for an existing repo', async () => {
    const r = await getGithubRepoStars({ owner: 'apify', repo: 'apify-sdk-js' });
    expect(r.found).toBe(true);
    expect(r.stars).toBeGreaterThan(0);
  });
  it('returns found:false for a missing repo', async () => {
    const r = await getGithubRepoStars({ owner: 'this-does-not-exist-xyz', repo: 'nope' });
    expect(r.found).toBe(false);
  });
});
```

### Integration tests (MCP Inspector)

```bash
APIFY_META_ORIGIN=STANDBY ACTOR_WEB_SERVER_PORT=8080 apify run -p
# In another terminal:
npx @modelcontextprotocol/inspector
# Set URL to http://localhost:8080/mcp, click Connect
# Run tools/list, invoke each tool, check the server logs
```

`Actor.charge()` is a no-op locally (logs only). Charges fire only on the Apify platform.

### End-to-end (deployed Actor)

After `apify push`:

```bash
curl -i \
  -H "Authorization: Bearer $APIFY_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}' \
  https://<your-username>--github-stars-mcp.apify.actor/mcp
```

Should return a 200 with the JSON-RPC response listing your tools.

Then point MCP Inspector at the production URL with your token. Confirm every tool can be invoked.

## What this file does NOT cover

- **PPE event taxonomy design** (which tools to map to which events, free-tier handling, idle-cost trade-offs) → your Apify monetization/PPE reference
- **README, Store description, input/dataset schemas for MCP variants** → your Apify Actor content/README reference
- **Crawlee/Playwright inside MCP tools (for tools that scrape)** → your scraping reference + this skill's cold-start guidance
- **Transports deep-dive (Streamable HTTP vs Legacy SSE vs stdio)** → `references/transports.md`
- **Path A proxy mode** → `references/path-a-proxy.md`
- **Publish checklist** → `references/checklist-publish.md`
