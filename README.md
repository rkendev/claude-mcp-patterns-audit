# claude-mcp-patterns-audit

[![links](https://github.com/rkendev/claude-mcp-patterns-audit/actions/workflows/links.yml/badge.svg)](https://github.com/rkendev/claude-mcp-patterns-audit/actions/workflows/links.yml)
[![markdownlint](https://github.com/rkendev/claude-mcp-patterns-audit/actions/workflows/markdownlint.yml/badge.svg)](https://github.com/rkendev/claude-mcp-patterns-audit/actions/workflows/markdownlint.yml)

Curated audit of 13 production MCP servers — what canonical CCA-F D2 patterns ship in the wild, and what anti-patterns hide in plain sight.

Built as Artifact D of a Claude Certified Architect Foundations small-projects portfolio. Companion to:

- [claude-mcp-server-minimal](https://github.com/rkendev/claude-mcp-server-minimal) (Artifact A — minimal MCP server)
- [claude-tools-roundtrip-playground](https://github.com/rkendev/claude-tools-roundtrip-playground) (Artifact B — native `tools=` round-trip)
- [claude-tool-choice-modes](https://github.com/rkendev/claude-tool-choice-modes) (Artifact C — `tool_choice` modes)

## How to use this audit

Pick an MCP from the table below and read its dedicated annotation file. Each annotation cites file:line in the source — the dated commit reference at the top of every annotation lets you re-verify against the current code.

## Summary table

| MCP | Language | Transport | Error envelope (D2) | `additionalProperties:false` | "Do NOT use for X" | Anti-patterns observed | File |
|---|---|---|---|---|---|---|---|
| filesystem | TypeScript | stdio | No | Implicit (Zod via SDK) | No | 1 | [link](mcps/filesystem.md) |
| git | Python | stdio | No | No | No | 3 | [link](mcps/git.md) |
| memory | TypeScript | stdio | No | Implicit (Zod via SDK) | No | 2 | [link](mcps/memory.md) |
| fetch | Python | stdio | No | No | No | 2 | [link](mcps/fetch.md) |
| github-mcp-server | Go | stdio + Streamable HTTP | No | Partial | Yes | 1 | [link](mcps/github-mcp-server.md) |
| sentry-mcp | TypeScript | stdio + remote (Worker) | No — but typed class hierarchy | Implicit (Zod via SDK) | Yes | 0 | [link](mcps/sentry-mcp.md) |
| cloudflare-workers-observability | TypeScript | SSE + Streamable HTTP | No | Implicit (Zod via SDK) | No | 1 | [link](mcps/cloudflare-workers-observability.md) |
| stripe-mcp | TypeScript | stdio (proxy to remote) | No | Not observed (proxy) | Not observed (proxy) | 0 | [link](mcps/stripe-mcp.md) |
| notion-mcp-server | TypeScript | stdio + Streamable HTTP | No | Partial | No | 3 | [link](mcps/notion-mcp-server.md) |
| supabase-mcp | TypeScript | stdio | No | Implicit (Zod via SDK) | Yes | 0 | [link](mcps/supabase-mcp.md) |
| postgres-mcp (CrystalDBA) | Python | stdio | No | Implicit (Pydantic) | No | 1 | [link](mcps/postgres-mcp.md) |
| x-twitter-mcp-server | Python | stdio | No | Implicit (FastMCP) | Partial | 9 | [link](mcps/x-twitter-mcp-server.md) |
| mcp-claude-hackernews | TypeScript | stdio | No | No | No | 4 | [link](mcps/mcp-claude-hackernews.md) |

The "Anti-patterns observed" column counts the bullet points in the `## Anti-patterns observed` section of each annotation. A 0 means none were observed in the audit at the pinned commit; it does not mean the server is flawless.

## Cross-cutting findings

### What's universal

Three patterns hold across all 13 servers. **None ship the canonical CCA-F D2 envelope** — no MCP in the audit emits a flat `errorCategory` + `isRetryable` + `message` shape from a tool. **All 13 read auth and configuration from environment variables (or CLI flags) at process launch**, not on first tool call — even the multi-tenant designs do their token reading at startup. And **stdio remains the default transport everywhere except `cloudflare-workers-observability`** (the lone remote-only server); the four servers that support remote/HTTP transports all keep stdio as a co-equal entrypoint, threading the same tool factory through both.

### What's contested

Three patterns split, and the splits are informative. **"Do NOT use for X" anti-instructions are vendor-correlated**: only `sentry-mcp` ([`get-issue-details.ts:47-49`](mcps/sentry-mcp.md)), `github-mcp-server` ([`pullrequests.go:1133`](mcps/github-mcp-server.md)), and `supabase-mcp` ([`database-operation-tools.ts:161-163`](mcps/supabase-mcp.md)) ship them as a discipline — all three are major-vendor-published. Community servers and Anthropic's reference servers omit them; `x-twitter-mcp-server` is the cleanest demonstration, putting the warning on `delete_all_bookmarks` but missing it on the equally destructive `delete_tweet` and `post_tweet`. **Schema strictness is language-correlated**: TypeScript/Zod users get `additionalProperties: false` essentially for free (Zod's `z.object` is strict by default and the SDK lifts that into JSONSchema); Python/Pydantic users have to opt in with `ConfigDict(extra="forbid")` — and `git`, `fetch`, and `postgres-mcp` all skip it. **Polymorphic schemas are domain-correlated**: where the underlying API is polymorphic (Sentry's `assignedTo`, Notion's block types, Cloudflare's timeframe), MCPs faithfully reproduce `anyOf`/`oneOf`; where it isn't, they don't. `github-mcp-server` deliberately replaced unions with a `method`-enum dispatcher.

### The biggest anti-pattern in the wild: lossy error envelopes

**12 of 13 servers degrade error structure on the way out.** Tells observed: `fetch-mcp` collapses connection-timeout, 429, 503, and 404 into the same `INTERNAL_ERROR` `McpError` ([`server.py:127-133`](mcps/fetch.md)) — the one MCP whose entire purpose is HTTP and the one most damaged by missing retryable signal. `github-mcp-server` captures rich typed errors server-side ([`pkg/utils/result.go:26-35`](mcps/github-mcp-server.md)), then flattens them to plain text on the wire — status codes, rate-limit headers, and retryability are not surfaced to the agent. `cloudflare-workers-observability` returns stringified-JSON errors without `isError: true` ([`workers-observability.tools.ts:183-194`](mcps/cloudflare-workers-observability.md)), and its `error instanceof Error && error.message` idiom ([line 189](mcps/cloudflare-workers-observability.md)) emits the literal `false` when the caught value is not an `Error`. The exception is `sentry-mcp`'s HTTP-status-keyed class hierarchy ([`api-client/errors.ts:19-326`](mcps/sentry-mcp.md)), which gives consumers `isPermissionError()` / `isRateLimitError()` type guards — close cousins of an `errorCategory` field — and a transport-aware redaction layer that strips internals over HTTP. It is the audit's strongest finding for D2 envelope discipline, and even it does not emit a flat `isRetryable` field.

### What hiring conversations should test

Four questions a candidate could answer from the audit alone, no hand-holding required.

1. **Transport-aware error redaction.** Open [`mcps/sentry-mcp.md`](mcps/sentry-mcp.md). What does `formatServerConfigError` do differently over HTTP vs stdio? Why does that matter for a server that ships both transports?
2. **Read-only modes in practice.** Compare the read-only flag in [`supabase-mcp.md`](mcps/supabase-mcp.md) and the safety mode in [`postgres-mcp.md`](mcps/postgres-mcp.md). Both ship the feature — describe the difference in *how* they enforce it (hard gate vs. session-level forwarding) and what that implies for an LLM caller's mental model.
3. **Description-as-routing under load.** [`x-twitter-mcp-server.md`](mcps/x-twitter-mcp-server.md) and [`stripe-mcp.md`](mcps/stripe-mcp.md) both expose paid-API write surfaces. One has anti-routing guidance on every destructive op; the other has it on one. Read the audit, identify which is which, and explain what the inconsistency tells you about description-as-routing as a real-world technique.
4. **Envelope vs. typed hierarchy.** Make a 200-word case for whether a flat `errorCategory` envelope adds value over Sentry's typed-class approach, given what you read in [`mcps/sentry-mcp.md`](mcps/sentry-mcp.md). There is no right answer; we want to hear how the candidate frames the trade-off.

## Methodology

- Each MCP analyzed at a frozen commit (cited at the top of its annotation).
- Annotations are dated; re-verify against current code via the cited commit. Annotations may go stale; the dated-commit reference lets a reader re-verify.
- Source observations are cited at file:line — no behavior is asserted without evidence. Where evidence was unavailable (e.g. `stripe-mcp` is a proxy to a remote server whose schemas are not in the local repo), the field is recorded as "Not observed in the audit" rather than inferred.
- Anti-patterns are noted descriptively, not pejoratively. Real production code makes pragmatic trade-offs that look "wrong" out of context.
- Two of the 13 picks (`x-twitter-mcp-server`, `mcp-claude-hackernews`) were chosen specifically as low-star community servers, to test whether even simple cases get pattern hygiene right. They form the contrast against which the major-vendor patterns become visible.

## License

MIT — see [LICENSE](LICENSE).
