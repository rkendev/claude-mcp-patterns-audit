# stripe-mcp

> **Repo:** [stripe/ai](https://github.com/stripe/ai) (formerly `stripe/agent-toolkit`) · **Subpath:** `tools/modelcontextprotocol/` · **License:** MIT · **Last analyzed:** 2026-05-10 · **Commit:** [`b40276ed`](https://github.com/stripe/ai/tree/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol)

## What it does

Stripe's official MCP entry. The `@stripe/mcp` package is a thin **stdio-to-HTTP proxy** to the hosted server at `https://mcp.stripe.com` ([`src/index.ts:14`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/index.ts#L14)). Tool names (customers, products, invoices, refunds, subscriptions, disputes, docs search) are enumerated in [`manifest.json:32-56`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/manifest.json#L32-L56); schemas/descriptions are served remotely.

## Architecture summary

Repo renamed `stripe/agent-toolkit` → `stripe/ai`. TypeScript ([`package.json:42-59`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/package.json#L42-L59)). Stdio inbound ([`src/index.ts:32`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/index.ts#L32)); streamable-HTTP outbound to `mcp.stripe.com` ([`src/index.ts:40-43`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/index.ts#L40-L43)). The first stdio message is sniffed for `clientInfo.name` to enrich User-Agent ([`src/userAgent.ts:8-26`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/userAgent.ts#L8-L26)); frames then pipe bidirectionally. **No local tool registration**. `server.json` re-advertises the remote per the MCP registry schema ([`server.json:10-15`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/server.json#L10-L15)).

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? **No**
- **Pattern observed:** `throw new Error(...)` for arg/key validation ([`src/cli.ts:30-33`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/cli.ts#L30-L33), [`src/cli.ts:39-53`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/cli.ts#L39-L53)); `console.error` for transport faults ([`src/index.ts:50-56`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/index.ts#L50-L56)); top-level `main().catch` rethrows after logging ([`src/index.ts:107-110`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/index.ts#L107-L110)). Tool-call envelopes originate remotely.
- **Anti-patterns observed:** None observed in the audit.

## Configuration

- **`.mcp.json` shape:** `command: "npx"`, `args: ["-y", "@stripe/mcp", "--api-key=…"]` ([`README.md:30-38`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/README.md#L30-L38)); optional `--stripe-account=acct_*` for Connect ([`src/cli.ts:27-28`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/cli.ts#L27-L28)).
- **Env-var expansion:** `STRIPE_SECRET_KEY` is the fallback ([`src/cli.ts:38`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/cli.ts#L38)); DXT injects it via `${user_config.stripe_secret_key}` ([`manifest.json:27-29`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/manifest.json#L27-L29)). Legacy `STRIPE_API_KEY` is not read.
- **`allowed-tools` usage:** No. `--tools` was removed; permissions delegate to **Restricted API Key (RAK)** scopes ([`src/cli.ts:17-25`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/cli.ts#L17-L25)).

## Tool design quality

- **`additionalProperties: false`** on input schemas? **Not observed in the audit** — schemas are remote-only.
- **"Do NOT use for X"** anti-instructions? **Not observed in the audit** — descriptions live remotely; the local code only ships a tool-name list ([`manifest.json:32-56`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/manifest.json#L32-L56)).
- **Polymorphic shapes** (`anyOf`, discriminated unions)? **Not observed in the audit**.

## Notable design decisions (lessons to copy)

- **DXT-style `manifest.json`** declares the secret key `required: true` and marshals it into the child env ([`manifest.json:12-30`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/manifest.json#L12-L30)) — clients render config UI without bespoke code.
- **Auth-format gating** rejects non-`sk_`/`rk_` keys and warns toward RAKs ([`src/cli.ts:49-63`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/cli.ts#L49-L63)); tool-permission scoping is delegated to the RAK ([`src/cli.ts:20-23`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/cli.ts#L20-L23)).
- **Thin-proxy architecture** — local package is stdio glue; tool surface stays centrally maintained at `mcp.stripe.com` ([`src/index.ts:14`](https://github.com/stripe/ai/blob/b40276ed49c1db917fef6539ebbee38f8cc4e2af/tools/modelcontextprotocol/src/index.ts#L14)).

## Anti-patterns observed (lessons NOT to copy)

- None observed in the audit.

## Verification

```bash
git clone https://github.com/stripe/ai.git /tmp/stripe-mcp
cd /tmp/stripe-mcp
git checkout b40276ed49c1db917fef6539ebbee38f8cc4e2af
```

Then re-read the cited file:line locations under `tools/modelcontextprotocol/`.
