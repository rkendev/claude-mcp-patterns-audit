# filesystem (modelcontextprotocol/servers)

> **Repo:** [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) · **Subpath:** `src/filesystem/` · **License:** MIT · **Last analyzed:** 2026-05-10 · **Commit:** [`4503e2d1`](https://github.com/modelcontextprotocol/servers/tree/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem)

## What it does

Anthropic's reference filesystem MCP. Exposes ~16 tools to read, write, edit, search, and inspect files within a sandbox of allowed directories. Used as the first MCP most developers configure with Claude Desktop or Cursor; cited as the canonical example in the SDK docs.

## Architecture summary

stdio transport ([`index.ts:4`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L4)) over `McpServer` (the high-level SDK wrapper, not the lower-level `Server` class). Single TypeScript module with tool logic delegated to `lib.ts`. Allowed directories are configurable two ways: command-line args, or via the **MCP Roots protocol** at runtime (`oninitialized` handler at [`index.ts:731-754`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L731-L754)). Symlink-resolved on startup ([`index.ts:50-66`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L50-L66)) so `/tmp` and `/private/tmp` both match on macOS.

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** Raw `throw new Error("...")` for argument validation ([`index.ts:195`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L195)) and fatal-startup conditions ([`index.ts:749`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L749)). Top-level `runServer().catch(...)` exits the process with code 1 ([`index.ts:764-766`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L764-L766)).
- **Anti-patterns observed:** Silent error degradation in directory listing — failed `fs.stat` returns synthetic `{size: 0, mtime: new Date(0)}` rather than surfacing the error ([`index.ts:480-487`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L480-L487)). No retryable signal anywhere.

## Configuration

- **`.mcp.json` shape:** `command: "npx"`, `args: ["-y", "@modelcontextprotocol/server-filesystem", "<dir1>", "<dir2>", ...]`. Allowed directories passed positionally.
- **Env-var expansion:** None at the server itself; clients (Claude Desktop, etc.) handle `${VAR}` substitution before launch.
- **`allowed-tools` usage:** n/a — server publishes tools; client decides which to allow.

## Tool design quality

- **`additionalProperties: false`** on input schemas? **Implicit (via SDK)** — schemas declared as Zod field maps (e.g. [`index.ts:237`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L237), [`index.ts:255`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L255)); the MCP SDK's `registerTool` lifts these into JSONSchema with `additionalProperties` defaulted by Zod-to-JSONSchema (strict by default for `z.object`).
- **"Do NOT use for X"** anti-instructions in tool descriptions? **No** — descriptions are entirely positive ("Use this tool when you need to..." at [`index.ts:230-235`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L230-L235)). The closest negative is "Only works within allowed directories" — a constraint, not anti-routing.
- **Polymorphic shapes** (`anyOf`, discriminated unions)? **No** — flat schemas with optional fields and enums (e.g. [`index.ts:261`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L261) `z.enum(["image", "audio", "blob"])`).

## Notable design decisions (lessons to copy)

- **Tool annotations for client routing** — every read tool sets `annotations: { readOnlyHint: true }` ([`index.ts:220`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L220), and ~10 more sites). Lets the client filter destructive ops in restricted modes without parsing descriptions.
- **MCP Roots protocol integration** — server can be (re)configured by the *client* mid-session via the roots/list_changed notification ([`index.ts:718-728`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L718-L728)). Allows IDEs to scope filesystem access to the active workspace.
- **Symlink-resolved sandbox** — both original and `realpath`-resolved paths stored ([`index.ts:50-66`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L50-L66)) to defeat `/tmp` → `/private/tmp` path-validation bypass on macOS.

## Anti-patterns observed (lessons NOT to copy)

- **Inconsistent error handling** — same module mixes `throw`, silent-degradation ([`index.ts:480-487`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L480-L487)), and `console.error` (e.g. [`index.ts:721`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/filesystem/index.ts#L721)). No retryable hint anywhere despite obvious candidates (transient `EBUSY`, `EAGAIN`).

## Verification

To re-verify this annotation against current code:

```bash
git clone https://github.com/modelcontextprotocol/servers.git /tmp/filesystem-mcp
cd /tmp/filesystem-mcp
git checkout 4503e2d12b799448cd05f789dd40f9643a8d1a6c
```

Then re-read the cited file:line locations under `src/filesystem/`.
