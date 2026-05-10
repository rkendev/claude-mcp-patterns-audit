# fetch (modelcontextprotocol/servers)

> **Repo:** [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) · **Subpath:** `src/fetch/` · **License:** MIT · **Last analyzed:** 2026-05-10 · **Commit:** [`4503e2d1`](https://github.com/modelcontextprotocol/servers/tree/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch)

## What it does

Reference Python MCP that fetches a URL over HTTP and returns either a markdown-extracted (via `readabilipy` + `markdownify`) or raw representation. Honors `robots.txt` for autonomous fetches, supports pagination via `start_index`/`max_length`, and exposes both a `fetch` tool and a `fetch` prompt for user-initiated retrieval.

## Architecture summary

stdio transport ([`server.py:287`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L287)). Python, confirmed via [`pyproject.toml:6`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/pyproject.toml#L6) (`requires-python = ">=3.10"`). Uses the lower-level `mcp.server.Server` class with decorator-based registration: `@server.list_tools()` ([`server.py:197`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L197)), `@server.call_tool()` ([`server.py:223`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L223)), `@server.list_prompts()` / `@server.get_prompt()`. HTTP done via `httpx.AsyncClient` with a 30s timeout ([`server.py:125`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L125)). Single-tool input schema is generated from a Pydantic `Fetch` model ([`server.py:151-178`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L151-L178)) via `model_json_schema()` ([`server.py:205`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L205)).

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** All HTTP failures collapsed to `McpError(ErrorData(code=INTERNAL_ERROR, ...))`. The `httpx.HTTPError` catch at [`server.py:127-128`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L127-L128) wraps connect/timeout/transport failures identically to a `>=400` HTTP status at [`server.py:129-133`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L129-L133). Pydantic validation errors map to `INVALID_PARAMS` ([`server.py:227-228`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L227-L228)). No retryable flag, no distinction between 5xx/429/timeout (retryable) and 404/403 (not).
- **Anti-patterns observed:** Robots.txt fetch failure surfaces as `INTERNAL_ERROR` regardless of cause ([`server.py:82-86`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L82-L86)); `<error>...</error>` strings embedded in successful tool results for empty-pagination cases ([`server.py:242,246`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L242)).

## Configuration

- **`.mcp.json` shape:** `command: "uvx"`, `args: ["mcp-server-fetch"]` plus optional `--user-agent`, `--ignore-robots-txt`, `--proxy-url` ([`__init__.py:9-21`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/__init__.py#L9-L21)).
- **Env-var expansion:** None at the server.
- **`allowed-tools` usage:** n/a — server publishes tools.

## Tool design quality

- **`additionalProperties: false`** on input schemas? **No** — `Fetch` Pydantic model ([`server.py:151-178`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L151-L178)) does not set `model_config = ConfigDict(extra='forbid')`, so `model_json_schema()` emits no `additionalProperties: false`.
- **"Do NOT use for X"** anti-instructions? **No** — tool description ([`server.py:202-204`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L202-L204)) is a positive override ("this tool now grants you internet access").
- **Polymorphic shapes** (`anyOf`, discriminated unions)? **No** — single flat schema with scalar fields.

## Notable design decisions (lessons to copy)

- **Robots.txt enforcement on autonomous calls, with a separate user-agent for prompt-driven calls** ([`server.py:23-24`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L23-L24), gated at [`server.py:234-235`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L234-L235)).
- **Pagination via `start_index` + `max_length` with explicit continuation hint** echoed back into the response ([`server.py:252-254`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L252-L254)) — teaches the model how to resume.
- **Pydantic-generated input schema** via `model_json_schema()` ([`server.py:205`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L205)) keeps the validator and schema in one source of truth.

## Anti-patterns observed (lessons NOT to copy)

- **No retryable/non-retryable distinction on HTTP errors** — connection timeout, 429, 503, and 404 all collapse to the same `INTERNAL_ERROR` `McpError` ([`server.py:127-133`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L127-L133)), forcing the LLM to guess whether a retry is sensible.
- **Schema not strict** — missing `extra='forbid'` on `Fetch` ([`server.py:151`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/fetch/src/mcp_server_fetch/server.py#L151)) silently ignores unknown args.

## Verification

To re-verify this annotation against current code:

```bash
git clone https://github.com/modelcontextprotocol/servers.git /tmp/fetch-mcp
cd /tmp/fetch-mcp
git checkout 4503e2d12b799448cd05f789dd40f9643a8d1a6c
```

Then re-read the cited file:line locations under `src/fetch/`.
