# MCP Transports — Choosing the Right One on Apify

Apify Standby supports three MCP transport options. The choice cascades into your client config, your `webServerMcpPath`, and (sometimes) compatibility with end-user MCP clients. This file is the canonical reference.

## The three transports

| Transport | Spec status | Apify `webServerMcpPath` | When to use |
|---|---|---|---|
| **Streamable HTTP** | Current default in the MCP spec | `/mcp` | All new MCP server Actors |
| **Legacy SSE** | Deprecated in spec, still supported | `/sse` | Only if a target client cannot do Streamable HTTP |
| **stdio** | Local-only by spec | N/A | Only inside Path A's proxy (Apify converts to HTTP) |

## Streamable HTTP — the default

**Use this unless you have a specific reason not to.**

- Single HTTP endpoint, typically `POST /mcp`.
- The server may upgrade the response to a stream (chunked transfer or SSE-style) for long-running tool calls and notifications.
- Stateless-friendly: no session state required server-side for the common per-tool-call pattern.
- Compatible with Apify's load balancer — multiple Standby containers can serve requests interchangeably.

### Server-side setup (Node)

```typescript
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';

app.post('/mcp', async (req, res) => {
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,  // stateless
  });
  await mcp.connect(transport);
  await transport.handleRequest(req, res, req.body);
});
```

### Server-side setup (Python / FastMCP)

```python
await mcp.run_streamable_http_async(host='0.0.0.0', port=port, path='/mcp')
```

### `.actor/actor.json`

```json
{
  "usesStandbyMode": true,
  "webServerMcpPath": "/mcp"
}
```

### Client config (Claude Desktop, Claude Code, Cursor, etc.)

```json
{
  "mcpServers": {
    "my-mcp": {
      "url": "https://<username>--<actor-name>.apify.actor/mcp",
      "headers": { "Authorization": "Bearer ${APIFY_API_TOKEN}" }
    }
  }
}
```

## Legacy SSE — fallback only

Some older MCP clients (and a handful of recent ones with slow update cycles) only support the deprecated SSE transport. Use this **only** when you've confirmed a target client requires it.

- The MCP spec marks SSE as legacy. New deployments should target Streamable HTTP.
- SSE has known issues with proxies and CDNs that buffer responses. Apify's edge does not buffer SSE, but downstream corporate proxies sometimes do.
- Bidirectional: client → server messages over POST, server → client over SSE stream.

### Server-side setup (Node)

```typescript
import { SSEServerTransport } from '@modelcontextprotocol/sdk/server/sse.js';

app.get('/sse', async (req, res) => {
  const transport = new SSEServerTransport('/messages', res);
  await mcp.connect(transport);
});
app.post('/messages', (req, res) => {
  // handled by the open transport, but you need an endpoint for incoming messages
});
```

### `.actor/actor.json`

```json
{
  "usesStandbyMode": true,
  "webServerMcpPath": "/sse"
}
```

The `webServerMcpPath` field tells Apify's MCP discovery (and any "Try this MCP server" UI in the Console) where to look. If you serve SSE at a different path, set this accordingly.

## stdio — only inside Path A proxies

A pure stdio MCP server cannot be deployed directly to Apify. Apify exposes HTTPS, not stdio. **However**, Path A's `ts-mcp-proxy` template can launch a stdio MCP server as a child process and translate its messages to Streamable HTTP:

```typescript
const MCP_COMMAND = ['npx', '@modelcontextprotocol/server-everything'];
// or
const MCP_COMMAND = ['python', '-m', 'my_stdio_mcp_module'];
```

The proxy spawns the child, reads/writes its stdin/stdout, and exposes the conversation over HTTP. From the deployed Actor's perspective, the transport is still Streamable HTTP — the stdio is internal to the container.

See `path-a-proxy.md` for the full pattern.

## Comparison

| Concern | Streamable HTTP | Legacy SSE | stdio (proxied) |
|---|---|---|---|
| Spec status | Current | Deprecated | Local-only |
| Apify Standby compatibility | Native | Native | Via proxy template |
| Client compatibility | All modern clients | Older clients | Same as Streamable HTTP after proxy |
| Stateful sessions | Optional (session ID) | Required (SSE channel state) | Inherited from child process |
| Behind corporate proxies | Works | Sometimes buffered/broken | Works |
| Apify load balancing | Stateless mode works | Session affinity needed | Stateless mode works |
| Debug with MCP Inspector | Direct | Direct | Direct (after proxy) |
| Cold-start impact | Standard | Standard | +child-process startup |

## Picking when uncertain

- **Default to Streamable HTTP.** It works with every modern MCP client and is the path Apify's own templates target.
- **Switch to SSE only if** you've tested with the target client and confirmed Streamable HTTP doesn't work.
- **Use the stdio proxy when** you have a working local stdio MCP server (yours or third-party) and just need to host it.

If you're building a new MCP server from scratch with no specific client constraints, **always Streamable HTTP.**

## MCP Inspector — the universal testing client

`@modelcontextprotocol/inspector` (run via `npx @modelcontextprotocol/inspector`) is the official MCP testing UI. It speaks all three transports.

Local Streamable HTTP:
```
URL: http://localhost:8080/mcp
Transport: Streamable HTTP
```

Local SSE:
```
URL: http://localhost:8080/sse
Transport: SSE
```

Remote (after `apify push`):
```
URL: https://<username>--<actor-name>.apify.actor/mcp
Transport: Streamable HTTP
Headers: { "Authorization": "Bearer <APIFY_API_TOKEN>" }
```

Inspector shows the raw JSON-RPC frames in both directions — invaluable for debugging tool registration, schema mismatches, and charge logging.

## Mixing transports

You can serve both Streamable HTTP and SSE from the same Actor. Useful if you want to support both new and legacy clients:

```typescript
// Streamable HTTP
app.post('/mcp', async (req, res) => { /* Streamable HTTP transport */ });

// SSE
app.get('/sse', async (req, res) => { /* SSE transport */ });
app.post('/messages', (req, res) => { /* SSE incoming messages */ });
```

But `webServerMcpPath` is a single value. Set it to your **primary** transport path (`/mcp`). The other transport is reachable by direct URL but won't be auto-discovered by Apify's MCP-aware tooling. This is rarely needed in practice — pick one transport.

## Common transport mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `webServerMcpPath: "/sse"` but server serves Streamable HTTP at `/mcp` | Apify MCP discovery fails; client connections return 404 | Match the path to the actual route |
| Trying to deploy a stdio-only MCP server | Apify build fails or container has nothing listening on `ACTOR_WEB_SERVER_PORT` | Use Path A's `ts-mcp-proxy` template instead |
| SSE behind a corporate proxy without disabling buffering | Client connects but never receives server messages | Set `Cache-Control: no-cache, no-transform` and `X-Accel-Buffering: no` headers; advise users to whitelist the Apify domain |
| `sessionIdGenerator` set when load balancing across multiple Standby containers | Some tool calls succeed, others fail with "session not found" | Set `sessionIdGenerator: undefined` (stateless) unless you need server-side session state |
| MCP Inspector configured with wrong transport | Connects but `tools/list` returns nothing | Match Inspector's transport selector to your server's transport |
