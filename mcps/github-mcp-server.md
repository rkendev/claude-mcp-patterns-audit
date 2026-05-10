# github-mcp-server

> **Repo:** [github/github-mcp-server](https://github.com/github/github-mcp-server) · **License:** MIT · **Last analyzed:** 2026-05-10 · **Commit:** [`0e2fc388`](https://github.com/github/github-mcp-server/tree/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4)

## What it does

GitHub's official MCP server: a Go binary exposing the GitHub REST + GraphQL API surface (issues, PRs, repos, code search, Actions, Copilot, Dependabot, code/secret scanning, gists, projects, discussions, notifications) as ~100 MCP tools. Distributed as a Docker image at `ghcr.io/github/github-mcp-server` and as a remote hosted endpoint with OAuth.

## Architecture summary

Dual transport: stdio ([`cmd/github-mcp-server/main.go:31-99`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/cmd/github-mcp-server/main.go#L31-L99)) and streamable HTTP ([`cmd/github-mcp-server/main.go:103-152`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/cmd/github-mcp-server/main.go#L103-L152)). Built on `github.com/modelcontextprotocol/go-sdk/mcp` ([`go.mod`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/go.mod)). Tools are organized into named **toolsets** declared as `ToolsetMetadata` constants ([`pkg/github/tools.go:21-167`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/github/tools.go#L21-L167)) — `context`, `repos`, `issues`, `pull_requests`, `actions`, `code_security`, `dependabot`, `notifications`, etc. `AllTools` returns every `ServerTool` tagged with its toolset ([`pkg/github/tools.go:174-318`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/github/tools.go#L174-L318)); an `Inventory` builder filters by enabled toolsets, read-only, exclude-tools, and PAT scope before calling `s.AddTool` ([`pkg/inventory/server_tool.go:110-119`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/inventory/server_tool.go#L110-L119)). A `dynamic` toolset exposes meta-tools (`enable_toolset`, `get_toolset_tools`) so the model can opt into surfaces at runtime ([`pkg/github/dynamic_tools.go:60-79`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/github/dynamic_tools.go#L60-L79)).

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** Domain-typed errors `GitHubAPIError` / `GitHubGraphQLError` / `GitHubRawAPIError` ([`pkg/errors/error.go:13-64`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/errors/error.go#L13-L64)) accumulate in a context bucket for middleware/observability; the surfaced tool result is just `IsError: true` with a text message via `NewToolResultErrorFromErr` ([`pkg/utils/result.go:26-35`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/utils/result.go#L26-L35), called from [`pkg/errors/error.go:157-163`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/errors/error.go#L157-L163)).
- **Anti-patterns observed:** No retryable signal exposed to the client even though HTTP status from `*github.Response` is available; rate-limit/transient hints are not propagated.

## Configuration

- **`.mcp.json` shape:** `command: "docker"`, `args: ["run","-i","--rm","-e","GITHUB_PERSONAL_ACCESS_TOKEN","ghcr.io/github/github-mcp-server"]`, `env: { GITHUB_PERSONAL_ACCESS_TOKEN: "${input:github_token}" }` ([`README.md:255-265`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/README.md#L255-L265)).
- **Env-var expansion:** Viper reads `GITHUB_*` env via `SetEnvPrefix("github") + AutomaticEnv` ([`cmd/github-mcp-server/main.go:209-214`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/cmd/github-mcp-server/main.go#L209-L214)); the PAT is mandatory — startup fails fast if unset ([`cmd/github-mcp-server/main.go:36-39`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/cmd/github-mcp-server/main.go#L36-L39)).
- **`allowed-tools` usage:** Server-side equivalents are richer than the client's `allowed-tools`: `--toolsets`, `--tools`, `--exclude-tools`, `--read-only`, `--dynamic-toolsets` ([`cmd/github-mcp-server/main.go:164-169`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/cmd/github-mcp-server/main.go#L164-L169)). Hosted/remote OAuth path also filters by token scopes ([`internal/ghmcp/server.go:149-152`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/internal/ghmcp/server.go#L149-L152)).

## Tool design quality

- **`additionalProperties: false`** on input schemas? **Partial** — applied on a nested object schema (`push_files.files[]` items) at [`pkg/github/repositories.go:1276`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/github/repositories.go#L1276); top-level tool schemas in the toolsnaps fixtures do not set it (only 1 of 102 `__toolsnaps__` files contains the keyword).
- **"Do NOT use for X"** anti-instructions? **Yes** — e.g. `list_pull_requests` description: "If the user specifies an author, then DO NOT use this tool and use the search_pull_requests tool instead." ([`pkg/github/pullrequests.go:1133`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/github/pullrequests.go#L1133)).
- **Polymorphic shapes** (`anyOf`, discriminated unions)? **No** — `grep` for `AnyOf`/`OneOf` in `pkg/github/*.go` returns no hits; consolidated tools (e.g. `issue_read`) use a `method` enum dispatcher instead ([`pkg/github/__toolsnaps__/issue_read.snap:13-22`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/github/__toolsnaps__/issue_read.snap#L13-L22)).

## Notable design decisions (lessons to copy)

- **Toolset modularization** — every tool factory tags itself with a `ToolsetMetadata` constant via the generic `NewTool[In, Out]` helper ([`pkg/github/dependencies.go:227-240`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/github/dependencies.go#L227-L240)); the inventory builder centralizes filtering by toolset / read-only / scope ([`internal/ghmcp/server.go:140-152`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/internal/ghmcp/server.go#L140-L152)). Lets a 100-tool surface ship without flooding the model's context.
- **PAT scope-derived tool filtering** — `RequiredScopes` on each tool are auto-expanded to `AcceptedScopes`, then matched against the live token's scopes fetched from the API ([`pkg/github/dependencies.go:237-238`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/github/dependencies.go#L237-L238), [`internal/ghmcp/server.go:262-269`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/internal/ghmcp/server.go#L262-L269)).
- **Dynamic toolset discovery** — `enable_toolset` lets the model expand its own surface when needed ([`pkg/github/dynamic_tools.go:60-79`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/github/dynamic_tools.go#L60-L79)).

## Anti-patterns observed (lessons NOT to copy)

- **Lossy error envelope** — rich typed errors (`GitHubAPIError` with `*github.Response`) are captured server-side but flattened to plain `IsError: true` text on the wire ([`pkg/errors/error.go:157-163`](https://github.com/github/github-mcp-server/blob/0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4/pkg/errors/error.go#L157-L163)); status code, rate-limit headers, and retryability are not surfaced to the agent.

## Verification

```bash
git clone https://github.com/github/github-mcp-server.git /tmp/github-mcp
cd /tmp/github-mcp
git checkout 0e2fc3889664c5b5bb770fd0ecfa4f6ca618f6b4
```

Then re-read the cited file:line locations.
