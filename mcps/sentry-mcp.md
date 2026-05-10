# sentry-mcp

> **Repo:** [getsentry/sentry-mcp](https://github.com/getsentry/sentry-mcp) · **License:** FSL-1.1-Apache-2.0 ([`LICENSE.md`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/LICENSE.md)) · **Last analyzed:** 2026-05-10 · **Commit:** [`0f4dd5a6`](https://github.com/getsentry/sentry-mcp/tree/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5)

## What it does

Sentry's official MCP server. Exposes ~30 tools for searching issues/events, analyzing traces, fetching docs, running Seer root-cause analysis, and managing projects/teams. Distributed both as an npm CLI (`@sentry/mcp-server`, stdio) and as a hosted Cloudflare Worker (remote/HTTP transport with OAuth). Heavily used inside Sentry; the canonical example of an observability-vendor MCP.

## Architecture summary

pnpm + turbo monorepo. `mcp-core` ([`packages/mcp-core/src/server.ts:22`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/server.ts#L22)) holds the transport-agnostic `buildServer()`, tools, error hierarchy, and Zod schemas. `mcp-server` is the stdio CLI: entry [`packages/mcp-server/src/index.ts:23-353`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-server/src/index.ts#L23-L353), transport at [`transports/stdio.ts:21`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-server/src/transports/stdio.ts#L21). `mcp-cloudflare` is the remote variant: OAuth provider + Hono app at [`packages/mcp-cloudflare/src/server/index.ts:1-26`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-cloudflare/src/server/index.ts#L1-L26). Both consume the same `buildServer()`. Transport mode is threaded through `ServerContext.transport` ([`mcp-core/src/server.ts:402`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/server.ts#L402)) so error formatting can suppress server-side details over HTTP.

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No** — but the strongest categorized hierarchy in the audit.
- **Pattern observed:** Class-based hierarchy keyed off HTTP status. `ApiError` base with `ApiClientError` (4xx, not reported to Sentry) and `ApiServerError` (5xx, reported); specialized subclasses `ApiPermissionError`/403, `ApiNotFoundError`/404, `ApiValidationError`/400|422, `ApiAuthenticationError`/401, `ApiRateLimitError`/429 (carries `retryAfter`) — all defined at [`mcp-core/src/api-client/errors.ts:19-242`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/api-client/errors.ts#L19-L242). Factory `createApiError` dispatches on status ([`errors.ts:249-326`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/api-client/errors.ts#L249-L326)). Domain errors `UserInputError`, `ConfigurationError`, `LLMProviderError` at [`errors.ts:5-34`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/errors.ts#L5-L34) signal "do not log to Sentry, surface to user." Tool failures are caught and converted to `{ isError: true, content: [...] }` ([`mcp-core/src/server.ts:384-407`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/server.ts#L384-L407)) — never thrown across the MCP boundary. `isApiAuthenticationErrorDeep` walks `cause` chains ([`errors.ts:335-343`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/api-client/errors.ts#L335-L343)). Retry policy is local to `retryWithBackoff` via a caller-supplied `shouldRetry` predicate ([`mcp-core/src/internal/fetch-utils.ts:43-75`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/internal/fetch-utils.ts#L43-L75)) — no `isRetryable` field is emitted in the envelope itself.
- **Anti-patterns observed:** None observed.

## Configuration

- **`.mcp.json` shape:** stdio: `command: "npx"`, `args: ["@sentry/mcp-server", "--access-token=...", "--host=sentry.io"]` (CLI flags parsed at [`cli/parse.ts`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-server/src/cli/parse.ts)). Remote: clients connect to `${baseUrl}/mcp` over OAuth ([`mcp-cloudflare/src/server/app.ts:28-37`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-cloudflare/src/server/app.ts#L28-L37)), optionally scoped `${baseUrl}/mcp/{org}/{project}`.
- **Env-var expansion:** `SENTRY_ACCESS_TOKEN`, `SENTRY_URL`, `SENTRY_HOST`, `MCP_URL`, `SENTRY_DSN`, `OPENAI_MODEL`, `ANTHROPIC_MODEL`, `EMBEDDED_AGENT_PROVIDER`, `MCP_SKILLS`, `MCP_DISABLE_SKILLS` — read at [`cli/parse.ts:79-95`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-server/src/cli/parse.ts#L79-L95). CLI flags win over env ([`parse.ts:98-108`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-server/src/cli/parse.ts#L98-L108)).
- **`allowed-tools` usage:** Skills-based gating via `MCP_SKILLS` / `--agent` mode ([`index.ts:286-292`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-server/src/index.ts#L286-L292)).

## Tool design quality

- **`additionalProperties: false`** on input schemas? **Implicit (via SDK)** — Zod field maps registered through `defineTool` (e.g. [`get-issue-details.ts:78-84`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/tools/get-issue-details.ts#L78-L84)); the MCP SDK lifts strictness from `z.object`.
- **"Do NOT use for X"** anti-instructions? **Yes** — `get_issue_details` description has explicit `"DO NOT USE for: - General searching..."` block ([`get-issue-details.ts:47-49`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/tools/get-issue-details.ts#L47-L49)), with `TRIGGER PATTERNS` and `<examples>`/`<hints>` sections.
- **Polymorphic shapes** (`anyOf`, discriminated unions)? **Yes** — `AssignedToSchema = z.union([z.null(), z.string(), z.object({type: z.enum(["user","team"]), ...}).passthrough()])` ([`api-client/schema.ts:324-335`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/api-client/schema.ts#L324-L335)); `z.union` heavily used for `id: string|number` polymorphism throughout `schema.ts`.

## Notable design decisions (lessons to copy)

- **HTTP-status-keyed error hierarchy** — `createApiError` at [`api-client/errors.ts:249-326`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/api-client/errors.ts#L249-L326) gives every consumer a typed branch (`isPermissionError()`, `isRateLimitError()`, etc.) without re-parsing status codes — close cousin of the `errorCategory` envelope.
- **Transport-aware error redaction** — `formatServerConfigError` at [`internal/error-handling.ts:81-91`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/internal/error-handling.ts#L81-L91) returns generic copy over HTTP, full details over stdio.
- **Errors-as-content invariant** — comment at [`mcp-core/src/server.ts:384-396`](https://github.com/getsentry/sentry-mcp/blob/0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5/packages/mcp-core/src/server.ts#L384-L396) explicitly forbids `throw` across the tool boundary; all errors return as `isError: true` text content.
- **Single `buildServer()` over two transports** — same factory consumed by stdio CLI and Cloudflare Worker.

## Anti-patterns observed (lessons NOT to copy)

- None observed in the audit.

## Verification

```bash
git clone https://github.com/getsentry/sentry-mcp.git /tmp/sentry-mcp
cd /tmp/sentry-mcp
git checkout 0f4dd5a68958fb946fa3a3cb1b79b838e5c05dd5
```

Then re-read the cited file:line locations.
