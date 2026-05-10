# cloudflare-workers-observability

> **Repo:** [cloudflare/mcp-server-cloudflare](https://github.com/cloudflare/mcp-server-cloudflare) · **Subpath:** `apps/workers-observability/` · **License:** Apache-2.0 · **Last analyzed:** 2026-05-10 · **Commit:** [`10a47554`](https://github.com/cloudflare/mcp-server-cloudflare/tree/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability)

## What it does

Remote MCP exposing Cloudflare Workers Observability: query logs, list available log keys, list values for a key. Hosted by Cloudflare at `observability.mcp.cloudflare.com` ([`apps/workers-observability/wrangler.jsonc:91`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/wrangler.jsonc#L91)). Three observability tools plus shared account/workers/docs tools registered from `@repo/mcp-common`.

## Architecture summary

Worker-as-MCP-host. The Worker's `fetch` handler ([`apps/workers-observability/src/workers-observability.app.ts:135-161`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/workers-observability.app.ts#L135-L161)) wraps the MCP server in `@cloudflare/workers-oauth-provider`'s `OAuthProvider`, exposing both `/mcp` (Streamable HTTP) and `/sse` (SSE) via `ObservabilityMCP.serve('/mcp')` / `serveSSE('/sse')`. `ObservabilityMCP` extends `McpAgent` from the `agents/mcp` package ([`workers-observability.app.ts:41`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/workers-observability.app.ts#L41)) — a Durable Object class (`MCP_OBJECT` binding, [`wrangler.jsonc:18-21`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/wrangler.jsonc#L18-L21)) that persists per-session state. `init()` builds a `CloudflareMCPServer` and registers tools ([`workers-observability.app.ts:54-92`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/workers-observability.app.ts#L54-L92)). Active account is cached in a separate `UserDetails` Durable Object ([`workers-observability.app.ts:103`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/workers-observability.app.ts#L103)). API-token bearer auth is short-circuited before the OAuth path ([`workers-observability.app.ts:136-138`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/workers-observability.app.ts#L136-L138)).

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** Tool handlers `try/catch` and return `{ content: [{ type: 'text', text: JSON.stringify({ error: '...' }) }] }` without `isError: true` ([`workers-observability.tools.ts:183-194`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/tools/workers-observability.tools.ts#L183-L194)). Missing-account is a plain text content message ([`workers-observability.tools.ts:64-74`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/tools/workers-observability.tools.ts#L64-L74)).
- **Anti-patterns observed:** `error instanceof Error && error.message` ([`workers-observability.tools.ts:189`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/tools/workers-observability.tools.ts#L189)) emits the literal `false` when `error` is not an `Error`. No `isError` flag, no retryable signal.

## Configuration

- **`.mcp.json` shape:** Remote MCP — clients connect to `https://observability.mcp.cloudflare.com/sse` or `/mcp`; no `command`/`args`. Routes pinned in [`wrangler.jsonc:91`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/wrangler.jsonc#L91).
- **Env-var expansion:** None client-side. Server reads Worker bindings: `OAUTH_KV`, `MCP_OBJECT`, `USER_DETAILS`, `MCP_METRICS`, `CLOUDFLARE_CLIENT_ID/SECRET` ([`wrangler.jsonc:14-58`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/wrangler.jsonc#L14-L58)).
- **`allowed-tools` usage:** n/a (server-side).

## Tool design quality

- **`additionalProperties: false`** on input schemas? **Implicit (via SDK)** — Zod field maps passed to `agent.server.tool` ([`workers-observability.tools.ts:54-56`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/tools/workers-observability.tools.ts#L54-L56)). No explicit `.strict()`.
- **"Do NOT use for X"** anti-instructions? **No** — descriptions are positive guidance ("Filtering Best Practices", "Troubleshooting" sections, [`workers-observability.tools.ts:32-52`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/tools/workers-observability.tools.ts#L32-L52)).
- **Polymorphic shapes** (`anyOf`, discriminated unions)? **Yes** — `zTimeframe = z.union([zTimeframeAbsolute, zTimeframeRelative])` ([`packages/mcp-common/src/types/workers-logs.types.ts:216`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/packages/mcp-common/src/types/workers-logs.types.ts#L216)); plain `z.union`, not `discriminatedUnion`.

## Notable design decisions (lessons to copy)

- **Worker-as-MCP-host** — the same Worker `fetch` exports both transports ([`workers-observability.app.ts:142-143`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/workers-observability.app.ts#L142-L143)) so legacy SSE clients and Streamable-HTTP clients hit one origin.
- **Dual auth** — OAuth 2.1 with PKCE via `OAuthProvider` for interactive clients; raw API-token bearer mode short-circuited at the top of `fetch` ([`workers-observability.app.ts:136-138`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/workers-observability.app.ts#L136-L138)). 1h access / 30d refresh TTLs ([`workers-observability.app.ts:155-156`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/workers-observability.app.ts#L155-L156)).
- **State in Durable Objects** — per-user `activeAccountId` survives session restarts via `UserDetails` DO ([`workers-observability.app.ts:103-104`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/workers-observability.app.ts#L103-L104)); MCP session itself is a DO (`new_sqlite_classes`, [`wrangler.jsonc:11`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/wrangler.jsonc#L11)).

## Anti-patterns observed (lessons NOT to copy)

- **Stringified-error envelope without `isError`** ([`workers-observability.tools.ts:184-193`](https://github.com/cloudflare/mcp-server-cloudflare/blob/10a47554af4d10d022e7ad75d83bfd012167fe35/apps/workers-observability/src/tools/workers-observability.tools.ts#L184-L193)) — clients can't distinguish failure from success without parsing the JSON text. The `error instanceof Error && error.message` idiom can serialize the boolean `false`.

## Verification

```bash
git clone https://github.com/cloudflare/mcp-server-cloudflare.git /tmp/cf-mcp
cd /tmp/cf-mcp
git checkout 10a47554af4d10d022e7ad75d83bfd012167fe35
```

Then re-read the cited file:line locations under `apps/workers-observability/`.
