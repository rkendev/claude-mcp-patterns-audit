# memory (modelcontextprotocol/servers)

> **Repo:** [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) · **Subpath:** `src/memory/` · **License:** MIT · **Last analyzed:** 2026-05-10 · **Commit:** [`4503e2d1`](https://github.com/modelcontextprotocol/servers/tree/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory)

## What it does

Reference knowledge-graph MCP. Stores `entities` (name, type, observations) and `relations` (from, to, relationType) in a single JSONL file ([`index.ts:50-65`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L50-L65)) and exposes 9 tools to create, delete, search, and read graph contents — giving Claude long-term memory across sessions.

## Architecture summary

stdio transport ([`index.ts:479-480`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L479-L480)) over the high-level `McpServer` SDK wrapper ([`index.ts:256-259`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L256-L259)). Single TypeScript module. Domain logic lives in `KnowledgeGraphManager` ([`index.ts:68-238`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L68-L238)); each operation reloads the JSONL file, mutates in memory, and rewrites the whole file ([`index.ts:101-117`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L101-L117)). Despite being a memory/resource-shaped server, **the MCP Resources primitive is not used** — `grep` for `Resource`/`registerResource`/`resources/list` returns no matches; the graph is exposed only via tools.

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** Raw `throw new Error(...)` for missing entities ([`index.ts:144`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L144)); `ENOENT` on the memory file is swallowed and treated as an empty graph ([`index.ts:93-98`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L93-L98)); top-level `main().catch` exits with code 1 ([`index.ts:484-487`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L484-L487)).
- **Anti-patterns observed:** No retryable signal on transient I/O; full file rewrite on every mutation has no concurrency guard.

## Configuration

- **`.mcp.json` shape:** `command: "npx"`, `args: ["-y", "@modelcontextprotocol/server-memory"]`; optional `env: { "MEMORY_FILE_PATH": "..." }`.
- **Env-var expansion:** `MEMORY_FILE_PATH` read directly ([`index.ts:15-19`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L15-L19)); relative paths resolved against the script directory; legacy `memory.json` auto-migrated to `memory.jsonl` ([`index.ts:26-43`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L26-L43)).
- **`allowed-tools` usage:** n/a — client-side concern.

## Tool design quality

- **`additionalProperties: false`** on input schemas? **Implicit (via SDK)** — schemas are Zod field maps (e.g. [`index.ts:267-269`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L267-L269)) lifted by the SDK.
- **"Do NOT use for X"** anti-instructions? **No** — descriptions are short and positive (e.g. [`index.ts:288`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L288)).
- **Polymorphic shapes** (`anyOf`, discriminated unions)? **No** — flat object schemas with arrays of homogeneous records ([`index.ts:243-253`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L243-L253)).

## Notable design decisions (lessons to copy)

- **Dual-channel responses** — every handler returns both human-readable `content` text and `structuredContent` matching a declared `outputSchema` ([`index.ts:270-279`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L270-L279)), letting clients consume either form.
- **JSONL persistence with backward-compat migration** — append-friendly line format plus one-shot `memory.json` -> `memory.jsonl` rename on first run ([`index.ts:26-43`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L26-L43)).
- **Search returns relations with at least one matching endpoint** ([`index.ts:200-204`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L200-L204)) so callers can discover edges to nodes outside the result set.

## Anti-patterns observed (lessons NOT to copy)

- **Full-file rewrite on every mutation** ([`index.ts:101-117`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/memory/index.ts#L101-L117)) with no locking — JSONL format is wasted; concurrent writers can corrupt the file.
- **Resources primitive not used** despite the server's data being inherently resource-shaped (entities have stable names) — clients cannot subscribe or list nodes through the standard MCP path.

## Verification

To re-verify this annotation against current code:

```bash
git clone https://github.com/modelcontextprotocol/servers.git /tmp/memory-mcp
cd /tmp/memory-mcp
git checkout 4503e2d12b799448cd05f789dd40f9643a8d1a6c
```

Then re-read the cited file:line locations under `src/memory/`.
