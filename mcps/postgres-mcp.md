# postgres-mcp (CrystalDBA)

> **Repo:** [crystaldba/postgres-mcp](https://github.com/crystaldba/postgres-mcp) · **License:** MIT · **Last analyzed:** 2026-05-10 · **Commit:** [`07eb329c`](https://github.com/crystaldba/postgres-mcp/tree/07eb329c8c48e49640e0d1b5b35465d4d024c3ee)

## What it does

Postgres performance-analysis MCP. Beyond `execute_sql`: index tuning (DTA + LLM), `EXPLAIN`, `pg_stat_statements` analytics, multi-check health tool.

## Architecture summary

Python 3.12, `mcp[cli]>=1.25.0` ([`pyproject.toml:7`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/pyproject.toml#L7)). `FastMCP` with `@mcp.tool` decorators ([`server.py:38`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L38)). Three transports via `--transport`: stdio, sse, streamable-http ([`server.py:568-574`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L568-L574)).

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** `format_error_response` returns plain `"Error: <str>"` ([`server.py:79-81`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L79-L81)); tools `try/except Exception` and stringify ([`server.py:111-113`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L111-L113)). `ErrorResult` has no category/retryable fields ([`artifacts.py:15-22`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/artifacts.py#L15-L22)).
- **Anti-patterns observed:** `obfuscate_password` applied only at startup ([`server.py:642`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L642)), not per-tool returns.

## Configuration

- **`.mcp.json` shape:** `command: "postgres-mcp"`, args `["--access-mode", "restricted|unrestricted", "<DATABASE_URL>"]`.
- **Env-var expansion:** `DATABASE_URI` read directly, falls back to positional arg ([`server.py:629`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L629)).
- **`allowed-tools` usage:** n/a — gating is server-side via access mode.

## Tool design quality

- **`additionalProperties: false`**? **Implicit (via SDK)** — params declared as Pydantic `Field(...)` ([`server.py:124-125`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L124-L125)); no `ConfigDict(extra='forbid')` observed. `@validate_call` on some tools ([`server.py:437`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L437)).
- **"Do NOT use for X"** anti-instructions? **No** — descriptions are positive; destructive `execute_sql` reads "Execute any SQL query" ([`server.py:610`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L610)).
- **Polymorphic shapes**? **No** — `Literal["dta", "llm"]` enums instead ([`server.py:440`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L440)).

## Notable design decisions (lessons to copy)

- **Two-mode access control, dynamic registration** — `AccessMode` enum ([`server.py:49-53`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L49-L53)); `execute_sql` registered with different description and annotations (`destructiveHint` vs `readOnlyHint`) per mode ([`server.py:607-624`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L607-L624)).
- **Defence-in-depth read-only** — `SafeSqlDriver` AST-validates via `pglast` against an allow-list ([`safe_sql.py:960-972`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/sql/safe_sql.py#L960-L972)), then forces `BEGIN TRANSACTION READ ONLY` regardless of caller flag ([`safe_sql.py:984-997`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/sql/safe_sql.py#L984-L997), [`sql_driver.py:230-231`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/sql/sql_driver.py#L230-L231)), plus 30s timeout ([`server.py:68`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L68)).
- **Per-tool `ToolAnnotations`** — read tools set `readOnlyHint=True` ([`server.py:86-89`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L86-L89)).

## Anti-patterns observed (lessons NOT to copy)

- **Default mode is `UNRESTRICTED`** ([`server.py:58`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L58), [`server.py:565`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L565)) — opt-in safety inverts the secure-default principle.
- **Mutable global `current_access_mode`** reassigned at runtime ([`server.py:603-604`](https://github.com/crystaldba/postgres-mcp/blob/07eb329c8c48e49640e0d1b5b35465d4d024c3ee/src/postgres_mcp/server.py#L603-L604)) couples policy to module state.

## Verification

```bash
git clone https://github.com/crystaldba/postgres-mcp.git /tmp/postgres-mcp
cd /tmp/postgres-mcp
git checkout 07eb329c8c48e49640e0d1b5b35465d4d024c3ee
```

Then re-read the cited file:line locations.
