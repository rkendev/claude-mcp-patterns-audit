# mcp-claude-hackernews

> **Repo:** [imprvhub/mcp-claude-hackernews](https://github.com/imprvhub/mcp-claude-hackernews) · **License:** MPL-2.0 · **Last analyzed:** 2026-05-10 · **Commit:** [`8a60c201`](https://github.com/imprvhub/mcp-claude-hackernews/tree/8a60c201385797e9e1978b38948c0d4b3e5a2aac)

## What it does

Community TypeScript MCP over the public Hacker News Firebase API. Five read-only tools (`hn_latest`, `hn_top`, `hn_best`, `hn_story`, `hn_comments`) returning plain-text-formatted lists and threads. No auth.

## Architecture summary

stdio transport ([`index.ts:446`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L446)) over the low-level `Server` class ([`index.ts:131-141`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L131-L141)), not `McpServer`. Single 460-line `index.ts`: an `axios`-based `HackerNewsAPI` class ([`index.ts:47`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L47)), hand-written JSONSchema tools list ([`index.ts:143-228`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L143-L228)), if-chain dispatcher ([`index.ts:230-389`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L230-L389)). Module-level mutable `lastStoriesList` ([`index.ts:129`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L129)) caches the last list so `hn_comments` accepts a 1-based `story_index`.

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** Raw `throw new Error("...")` for argument validation ([`index.ts:304`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L304), [`index.ts:334`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L334), [`index.ts:343`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L343), [`index.ts:384`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L384)). Outer `try` log-and-rethrow ([`index.ts:385-388`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L385-L388)).
- **Anti-patterns observed:** Network failures swallowed: every `HackerNewsAPI` method `console.error`s and returns `[]` or `null` ([`index.ts:55-58`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L55-L58), [`index.ts:91-94`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L91-L94), [`index.ts:105-107`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L105-L107)), so the model sees "No stories found." ([`index.ts:393`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L393)) on transient outages with no retryable hint.

## Configuration

- **`.mcp.json` shape:** `command: "node"`, `args: ["<path>/build/index.js"]` (per `bin` script at [`package.json:8-9`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/package.json#L8-L9)).
- **Env-var expansion:** None — no `process.env` reads anywhere in `index.ts`.
- **`allowed-tools` usage:** n/a — client-side concern.

## Tool design quality

- **`additionalProperties: false`** on input schemas? **No** — omitted on every tool ([`index.ts:149-160`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L149-L160), [`index.ts:197-206`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L197-L206)).
- **"Do NOT use for X"** anti-instructions? **No** — descriptions are 6-12 words, positive only ([`index.ts:148`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L148), [`index.ts:210`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L210)). No `annotations.readOnlyHint`.
- **Polymorphic shapes**? **No** — `hn_comments` takes two optional siblings `story_id`/`story_index` ([`index.ts:213-223`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L213-L223)), neither `required`, enforced only in handler logic.

## Notable design decisions (lessons to copy)

- None observed in the audit.

## Anti-patterns observed (lessons NOT to copy)

- **Schemas without `additionalProperties: false`** ([`index.ts:149-225`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L149-L225)).
- **Either/or args as two optionals** instead of `anyOf` discriminator ([`index.ts:213-223`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L213-L223)); validity enforced at runtime ([`index.ts:333-344`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L333-L344)).
- **Silent network-error degradation** returning `[]` ([`index.ts:55-58`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L55-L58)).
- **Module-level mutable session state** (`lastStoriesList`) shared across callers ([`index.ts:129`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L129), written [`index.ts:247`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L247)).
- **Terse tool descriptions**, no anti-routing ([`index.ts:148`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L148), [`index.ts:180`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L180)).
- **Missing `annotations.readOnlyHint`** ([`index.ts:145-226`](https://github.com/imprvhub/mcp-claude-hackernews/blob/8a60c201385797e9e1978b38948c0d4b3e5a2aac/index.ts#L145-L226)).

## Verification

```bash
git clone https://github.com/imprvhub/mcp-claude-hackernews.git /tmp/hn-mcp
cd /tmp/hn-mcp
git checkout 8a60c201385797e9e1978b38948c0d4b3e5a2aac
```

Then re-read the cited file:line locations.
