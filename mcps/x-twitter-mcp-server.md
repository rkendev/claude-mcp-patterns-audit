# x-twitter-mcp-server

> **Repo:** [rafaljanicki/x-twitter-mcp-server](https://github.com/rafaljanicki/x-twitter-mcp-server) · **License:** MIT · **Last analyzed:** 2026-05-10 · **Commit:** [`1dd8b568`](https://github.com/rafaljanicki/x-twitter-mcp-server/tree/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649)

## What it does

Community Python MCP wrapping Tweepy to expose ~24 X/Twitter tools: profile lookups, follower pagination, post/delete tweet, polls, likes, bookmarks (incl. bulk-delete), timeline, recent search, trends, mentions. Distributed via PyPI and Smithery; supports STDIO and Streamable HTTP.

## Architecture summary

Single 556-line module built on `FastMCP` ([`server.py:22`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L22)). Tools registered via `@server.tool(name=..., description=...)` with Python type hints; FastMCP derives the JSONSchema. Tweepy v2 `Client` and v1.1 `API` are lazy-initialized on first call ([`server.py:28-65`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L28-L65)). HTTP transport adds a Starlette ASGI wrapper ([`http_server.py:11-48`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/http_server.py#L11-L48)) and a Smithery per-request config middleware ([`middleware.py:9-51`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/middleware.py#L9-L51)).

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** raw `raise Exception("Tweet action rate limit exceeded")` ([`server.py:252`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L252)) and `raise EnvironmentError(...)` for missing creds ([`server.py:45`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L45)). Tweepy and `requests.raise_for_status()` errors propagate untouched ([`server.py:93`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L93), [`server.py:113`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L113)). No try/except around any tool body; FastMCP serialises whatever bubbles up.
- **Anti-patterns observed:** see dedicated section.

## Configuration

- **`.mcp.json` shape:** `command: "x-twitter-mcp-server"` (PyPI) or `command: "uv", args: ["--directory", ..., "run", "x-twitter-mcp-server"]` ([README:207-244](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/README.md#L207-L244)).
- **Env-var expansion:** `TWITTER_API_KEY`, `TWITTER_API_SECRET`, `TWITTER_ACCESS_TOKEN`, `TWITTER_ACCESS_TOKEN_SECRET`, `TWITTER_BEARER_TOKEN` read via `os.getenv` ([`server.py:36-54`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L36-L54)); `TWITTER_OAUTH2_USER_ACCESS_TOKEN` for bookmarks ([`server.py:84`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L84)). `.env` auto-loaded ([`server.py:19`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L19)).
- **`allowed-tools` usage:** not observed in the audit.

## Tool design quality

- **`additionalProperties: false`** on input schemas? **Implicit (via SDK)** — schemas are derived by FastMCP from Python type hints (e.g. [`server.py:242`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L242)); strictness depends on FastMCP defaults — not explicitly set in this repo.
- **"Do NOT use for X"** anti-instructions? **Partial** — present on `delete_all_bookmarks` ("DESTRUCTIVE AND IRREVERSIBLE… Always confirm explicitly with the user before calling this tool", [`server.py:383`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L383)) but absent on the equally destructive `delete_tweet`, whose description is just "Delete a tweet by its ID" ([`server.py:271`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L271)) and `post_tweet` ("Post a tweet with optional media, reply, and tags", [`server.py:241`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L241)).
- **Polymorphic shapes** (`anyOf`, discriminated unions)? **No** — flat `Optional[...]` parameters throughout.

## Notable design decisions (lessons to copy)

- **Strong anti-instruction on the irreversible bulk op:** `delete_all_bookmarks` description and docstring both spell out "IRREVERSIBLE" and require explicit user confirmation ([`server.py:383-393`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L383-L393)).
- **Lazy client init** with cached singletons avoids paying OAuth setup until first tool call ([`server.py:32-33`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L32-L33)).
- **Explicit `requests` timeout** of 30s on every bookmark call ([`server.py:67`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L67), [`server.py:112`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L112)).

## Anti-patterns observed (lessons NOT to copy)

- **Rate-limit overflows raise bare `Exception`** with no retryable signal ([`server.py:252`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L252), [`server.py:411`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L411)).
- **In-process `defaultdict` rate counter** that resets on restart and is not shared across HTTP workers ([`server.py:126`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L126)); the inline comment says "use Redis in production".
- **Inconsistent destructive-op warnings** — `delete_tweet`, `unfavorite_tweet`, `delete_bookmark` carry no anti-instructions ([`server.py:271`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L271), [`server.py:343`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L343), [`server.py:370`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L370)).
- **"Simulated" tools silently substitute different semantics** — `vote_on_poll` returns a fabricated `{"status": "voted"}` ([`server.py:328`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L328)); `get_user_followers_you_know` returns plain followers ([`server.py:222`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L222)); `get_user_subscriptions` returns the following list ([`server.py:237`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L237)); `get_highlights_tweets` returns the user timeline ([`server.py:535`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L535)).
- **Bare `except Exception` swallows config-decode errors** in the Smithery middleware, downgrading silently to empty config ([`middleware.py:30-31`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/middleware.py#L30-L31)).
- **Per-request mutation of `os.environ`** ([`middleware.py:46-49`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/middleware.py#L46-L49)) plus the cached client singletons ([`server.py:25-26`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L25-L26)) means request N's credentials can persist into request N+1.
- **CORS wildcard with credentials** — `allow_origins=["*"]` plus `allow_credentials=True` ([`http_server.py:38-40`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/http_server.py#L38-L40)).
- **No input bounds in schema** — tweet-text 280-char limit, poll 5–10080 minute window etc. live in docstrings only; runtime clamping is ad hoc ([`server.py:493-502`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L493-L502)).
- **Near-identical descriptions** on tools that return different data ([`server.py:178`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L178), [`server.py:208`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L208), [`server.py:224`](https://github.com/rafaljanicki/x-twitter-mcp-server/blob/1dd8b568fe63762a3c9fa4d4988c759b1c0e8649/src/x_twitter_mcp/server.py#L224)).

## Verification

```bash
git clone https://github.com/rafaljanicki/x-twitter-mcp-server.git /tmp/x-twitter-mcp
cd /tmp/x-twitter-mcp
git checkout 1dd8b568fe63762a3c9fa4d4988c759b1c0e8649
```

Then re-read the cited file:line locations.
