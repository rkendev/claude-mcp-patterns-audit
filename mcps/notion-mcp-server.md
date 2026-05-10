# notion-mcp-server

> **Repo:** [makenotion/notion-mcp-server](https://github.com/makenotion/notion-mcp-server) · **License:** MIT · **Last analyzed:** 2026-05-10 · **Commit:** [`3bef7add`](https://github.com/makenotion/notion-mcp-server/tree/3bef7addac59b237da3bb41f36a520babc47fa3c)

## What it does

Notion's official MCP. Auto-generates one tool per OpenAPI operation from the bundled Notion REST spec (v2025-09-03; 22 endpoints). Stdio and Streamable HTTP transports.

## Architecture summary

TypeScript on the low-level `Server` ([`proxy.ts:94`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/mcp/proxy.ts#L94)). `OpenAPIToMCPConverter` walks `scripts/notion-openapi.json`, emitting one tool per operationId ([`parser.ts:183-200`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/openapi/parser.ts#L183-L200)); calls dispatch through `openapi-client-axios` ([`http-client.ts:147-165`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/client/http-client.ts#L147-L165)). HTTP mode gated by an auto-generated bearer in a 0o600 tmp file ([`start-server.ts:90-95`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/scripts/start-server.ts#L90-L95)).

## Error handling

- **Canonical CCA-F D2 envelope**? **No**
- **Pattern observed:** `HttpClientError` reshaped into `{status:'error', ...data}` text content ([`proxy.ts:178-191`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/mcp/proxy.ts#L178-L191)); non-HTTP errors re-thrown.
- **Anti-patterns observed:** no `isRetryable` for 429/5xx; body stringified verbatim ([`proxy.ts:172`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/mcp/proxy.ts#L172)).

## Configuration

- **`.mcp.json` shape:** `command: "npx"`, `args: ["-y", "@notionhq/notion-mcp-server"]`, `env: { NOTION_TOKEN }`; HTTP mode adds `--transport http --port N`.
- **Env-var expansion:** `NOTION_TOKEN` (builds `Authorization: Bearer` + `Notion-Version: 2025-09-03`) or `OPENAPI_MCP_HEADERS` ([`proxy.ts:202-228`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/mcp/proxy.ts#L202-L228)). Single-tenant.
- **`allowed-tools` usage:** n/a.

## Tool design quality

- **`additionalProperties: false`**? **Partial** — propagated when set in the spec ([`parser.ts:142-148`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/openapi/parser.ts#L142-L148)); else `true`. Used on block-request shapes ([`notion-openapi.json:223`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/scripts/notion-openapi.json#L223)).
- **"Do NOT use for X"** anti-instructions? **No** — descriptions are OpenAPI summaries prefixed `"Notion | "` ([`parser.ts:562-567`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/openapi/parser.ts#L562-L567)) plus auto error-responses ([`parser.ts:441-452`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/openapi/parser.ts#L441-L452)).
- **Polymorphic shapes**? **Yes** — `parentRequest` is `oneOf` with `const` discriminators ([`notion-openapi.json:111-130`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/scripts/notion-openapi.json#L111-L130)); `blockObjectRequest` is `anyOf` over per-block schemas ([`notion-openapi.json:253-258`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/scripts/notion-openapi.json#L253-L258)); preserved by the converter ([`parser.ts:158-166`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/openapi/parser.ts#L158-L166)).

## Notable design decisions (lessons to copy)

- **OpenAPI as single source of truth** — one spec drives tools, schemas, and HTTP client ([`parser.ts:236-256`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/openapi/parser.ts#L236-L256)).
- **Annotations from HTTP verb** — GET ⇒ `readOnlyHint`, else `destructiveHint` ([`proxy.ts:128-141`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/mcp/proxy.ts#L128-L141)).
- **String-fallback wrapper** on complex params + recursive `deserializeParams` defend against client double-serialization ([`parser.ts:490-516`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/openapi/parser.ts#L490-L516), [`proxy.ts:33-84`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/mcp/proxy.ts#L33-L84)).
- **64-char name truncation + collision suffix** ([`parser.ts:547-559`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/openapi/parser.ts#L547-L559)).

## Anti-patterns observed (lessons NOT to copy)

- **Paginated results inlined as text** — `has_more`/`next_cursor` stringified into one `text` block ([`proxy.ts:168-175`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/mcp/proxy.ts#L168-L175)); MCP Resources unused.
- **Single-tenant bearer baked at construction** ([`proxy.ts:99-105`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/openapi-mcp-server/mcp/proxy.ts#L99-L105)); HTTP mode reuses it across sessions.
- **`process.exit(1)` from a library function** on spec-load failure ([`init-server.ts:23`](https://github.com/makenotion/notion-mcp-server/blob/3bef7addac59b237da3bb41f36a520babc47fa3c/src/init-server.ts#L23)).

## Verification

```bash
git clone https://github.com/makenotion/notion-mcp-server.git /tmp/notion-mcp
cd /tmp/notion-mcp
git checkout 3bef7addac59b237da3bb41f36a520babc47fa3c
```

Then re-read the cited file:line locations.
