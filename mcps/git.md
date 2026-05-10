# git (modelcontextprotocol/servers)

> **Repo:** [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers) · **Subpath:** `src/git/` · **License:** MIT · **Last analyzed:** 2026-05-10 · **Commit:** [`4503e2d1`](https://github.com/modelcontextprotocol/servers/tree/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git)

## What it does

Anthropic's reference Git MCP. Exposes 12 tools wrapping common Git operations (status, diff, commit, add, reset, log, branch create/list/checkout, show) over a GitPython-backed `Repo`. Lets an LLM inspect and mutate a local Git repo without shelling out to `git`.

## Architecture summary

stdio transport ([`server.py:6`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L6), invoked at [`server.py:586`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L586)) over the lower-level `Server` class ([`server.py:4`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L4)), not the high-level `McpServer`. Single Python module (~590 lines). Pydantic v2 models per tool ([`server.py:23-93`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L23-L93)) generate input schemas via `model_json_schema()`. Tool names are an `Enum` ([`server.py:96-109`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L96-L109)) dispatched through a `match` statement ([`server.py:482`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L482)). Deps: `gitpython`, `mcp`, `pydantic`, `click` ([`pyproject.toml:18-23`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/pyproject.toml#L18-L23)). CLI entry takes optional `--repository` to scope access ([`__init__.py:7-21`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/__init__.py#L7-L21)).

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** Raw `raise ValueError(...)` / `raise BadName(...)` / `raise TypeError(...)` for validation failures ([`server.py:124`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L124), [`server.py:247`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L247), [`server.py:583`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L583)). `server.run(..., raise_exceptions=True)` ([`server.py:587`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L587)) lets the SDK convert exceptions to JSON-RPC errors. No retryable hint.
- **Anti-patterns observed:** `git_branch` returns the validation failure as a *successful* text result instead of raising ([`server.py:285`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L285)) — inconsistent with all other tools.

## Configuration

- **`.mcp.json` shape:** `command: "uvx"` (or `python -m mcp_server_git`), `args: ["mcp-server-git", "--repository", "<path>"]`. Single optional repository scope, passed via `click` flag.
- **Env-var expansion:** None at the server; client handles substitution.
- **`allowed-tools` usage:** n/a — server publishes tools, client decides allow-list.

## Tool design quality

- **`additionalProperties: false`** on input schemas? **No** — Pydantic v2's `model_json_schema()` does not emit `additionalProperties: false` by default, and no `model_config = ConfigDict(extra="forbid")` is set on any of the 12 models ([`server.py:23-93`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L23-L93)).
- **"Do NOT use for X"** anti-instructions in tool descriptions? **No** — descriptions are short positive blurbs, e.g. `"Shows the working tree status"` ([`server.py:311`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L311)). Field descriptions on `GitBranch` include negative guidance like `"Do not pass anything to this param if no commit sha is specified"` ([`server.py:88`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L88)) but not at tool level.
- **Polymorphic shapes** (`anyOf`, discriminated unions)? **No** — flat Pydantic models with primitives and `Optional[str]`.

## Notable design decisions (lessons to copy)

- **Flag-injection defense in depth** — every tool that interpolates user strings into `git` CLI args rejects values starting with `-` before calling GitPython ([`server.py:123-124`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L123-L124), [`server.py:147-150`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L147-L150), [`server.py:202-203`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L202-L203), [`server.py:213`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L213)) and `git add` uses `--` to terminate option parsing ([`server.py:137`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L137)).
- **Sandboxed repo path with symlink resolution** — `validate_repo_path` resolves both sides and uses `Path.relative_to` to confirm containment ([`server.py:237-255`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L237-L255)), called on every tool invocation ([`server.py:477`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L477)).
- **`ToolAnnotations` on every tool** — `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint` set explicitly per tool ([`server.py:313-318`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L313-L318), [`server.py:379-384`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L379-L384)) so clients can filter destructive ops without parsing descriptions.

## Anti-patterns observed (lessons NOT to copy)

- **Mixed error-vs-success channel in `git_branch`** — invalid `branch_type` returns a normal text result rather than raising ([`server.py:285`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L285)), inconsistent with sibling tools that `raise BadName`.
- **Schemas not strict** — no `extra="forbid"` on any Pydantic model ([`server.py:23-93`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L23-L93)), so unknown args are silently ignored by Pydantic and absent from the published schema.
- **Tool descriptions lack negative guidance** — terse one-liners ([`server.py:311`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L311), [`server.py:432`](https://github.com/modelcontextprotocol/servers/blob/4503e2d12b799448cd05f789dd40f9643a8d1a6c/src/git/src/mcp_server_git/server.py#L432)) give no routing hints (e.g. when to prefer `git_diff` vs `git_show`).

## Verification

To re-verify this annotation against current code:

```bash
git clone https://github.com/modelcontextprotocol/servers.git /tmp/git-mcp
cd /tmp/git-mcp
git checkout 4503e2d12b799448cd05f789dd40f9643a8d1a6c
```

Then re-read the cited file:line locations under `src/git/`.
