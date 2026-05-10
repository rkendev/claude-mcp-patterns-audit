# T005 — Artifact D: MCP Resources Audit (curation repo)

## Goal

Ship the fourth small-projects-portfolio artifact: a public GitHub repo that audits **10–15 real production MCP servers** for canonical CCA-F D2 patterns and anti-patterns. Reader scrolls one summary table, picks an MCP that interests them, reads a 200–300-word deep-dive on its file:line decisions. The aggregate teaches "how do mature MCPs actually signal?" without us building one from scratch.

This is a NEW public repo with a fundamentally different shape than Artifacts A/B/C — markdown-heavy, no Python tests, no copier scaffold from `roy-ai-template`. Forcing it into the Python template would be cargo-culting; the artifact IS the curation.

Repo name: `claude-mcp-patterns-audit`
Local location on VPS: `/root/projects/AI-Engineering/short_projects_exam_prep/claude-mcp-patterns-audit/`

## Workflow

Three phases. Single feature branch `feat/t005-mcp-audit`. PR-based merge with explicit approval.

**Phase A — Setup (~1 hr):** create minimal repo structure (README placeholder, LICENSE, `.github/workflows/` for link check + markdownlint), public on GitHub, topics + secrets (none needed beyond `gh auth` itself).

**Phase B — Research & curation (~6–8 hrs):** identify 10–15 candidate MCPs (in plan mode, before Phase A starts). For each, write `mcps/<slug>.md` with the standardized audit template. Cite file:line where observations come from.

**Phase C — Synthesis (~1–2 hrs):** populate top-level README with summary table + cross-cutting findings (which patterns are universal, which are contested, which are anti-patterns). Add CHANGELOG. Open PR.

## Phase A — Setup

No `copier copy`. Build the repo by hand:

```bash
cd /root/projects/AI-Engineering/short_projects_exam_prep
mkdir -p claude-mcp-patterns-audit/{mcps,.github/workflows}
cd claude-mcp-patterns-audit

# Initialize git
git init && git checkout -b main

# License (MIT, matches A/B/C)
cat > LICENSE <<'EOF'
MIT License

Copyright (c) 2026 Roy Kensmil

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
EOF

# Minimal .gitignore
cat > .gitignore <<'EOF'
.DS_Store
*.swp
.idea/
.vscode/
EOF
```

### A.1 GitHub Actions workflows

Two workflows, both lightweight:

**`.github/workflows/links.yml`** — broken-link checker. Use `lycheeverse/lychee-action`:

```yaml
name: links

on:
  push:
    branches: [main]
  pull_request:

jobs:
  link-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Lychee link check
        uses: lycheeverse/lychee-action@v2
        with:
          args: --verbose --no-progress --max-redirects 5 './**/*.md'
          fail: true
```

**`.github/workflows/markdownlint.yml`** — markdown style consistency. Use `DavidAnson/markdownlint-cli2-action`:

```yaml
name: markdownlint

on:
  push:
    branches: [main]
  pull_request:

jobs:
  markdownlint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: DavidAnson/markdownlint-cli2-action@v17
        with:
          globs: '**/*.md'
```

Add a top-level `.markdownlint.json` with reasonable defaults:

```json
{
  "MD013": false,
  "MD033": { "allowed_elements": ["br", "details", "summary"] },
  "MD041": false
}
```

(Disable line-length, allow some HTML for collapsible sections, allow first-line-not-h1 since per-MCP files start with metadata blocks.)

### A.2 Repo + initial commit

```bash
git add -A
git commit -m "chore: initial scaffold — LICENSE, .github/workflows, .gitignore, .markdownlint.json"

gh repo create rkendev/claude-mcp-patterns-audit \
  --public \
  --source=. \
  --description "Curated audit of 10–15 production MCP servers — CCA-F D2 patterns and anti-patterns observed in the wild." \
  --push

gh repo edit rkendev/claude-mcp-patterns-audit \
  --add-topic mcp,model-context-protocol,claude,anthropic,audit,patterns,cca-f
```

### A.3 Phase A acceptance gates

- Public repo created; topics visible.
- Both workflows fire on initial push and pass green (no markdown to lint yet beyond LICENSE → both should pass).
- README does NOT yet exist (intentional — Phase C creates it).

## Phase B — Research & curation

### B.1 Candidate selection (do this in plan mode)

In plan mode, propose **10–15 MCPs** with one-line rationale per pick. Cover:

- **Anthropic-published examples** (any from `modelcontextprotocol/servers` repo — likely 4–6 candidates; pick the ones with substantive behavior, not toy examples).
- **Major-vendor servers**: Brave, Cloudflare, GitHub, Sentry, others if found.
- **Notable community servers** from `topic:model-context-protocol` filtered by stars (≥50 stars or active maintenance).
- **At least 2 deliberately-mediocre examples** for the anti-pattern lessons. Don't only audit best-in-class — the audit's value comes from contrast.

Selection criteria, in plan-mode output:

- Source visible (open-source, not just a binary distribution).
- Substantive enough to have an `errorCategory`-like decision somewhere (not 50-line toys).
- Last commit within 12 months (no abandoned drift).
- Different transport layers covered if possible (stdio, SSE, both) — the audit benefits from variety.

Wait for explicit approval on the candidate list before starting Phase B annotation work. **The list IS the artifact's quality gate.** A bad list = a bad audit.

### B.2 Per-MCP annotation template

For each accepted MCP, create `mcps/<slug>.md` (slug = lowercase-with-hyphens). Use this exact template:

```markdown
# <MCP Name>

> **Repo:** [<owner/name>](<github URL>) · **License:** <SPDX> · **Last analyzed:** <YYYY-MM-DD> · **Commit:** [`<short-sha>`](<github commit URL>)

## What it does

One paragraph (≤80 words). What problem does this MCP solve? Who uses it?

## Architecture summary

≤150 words. Transport (stdio / SSE / both), language, structure (single tool / multi-tool / resources / prompts), notable dependencies.

## Error handling

- **Canonical CCA-F D2 envelope** (`errorCategory` + `isRetryable` + `message`)? `<Yes / Partial / No>`
- **Pattern observed:** [verbatim quote or summary, with file:line citation]
- **Anti-patterns observed:** [if any, with citation]

## Configuration

- **`.mcp.json` shape:** [shape summary or "n/a — published as npm/pypi"]
- **Env-var expansion:** [pattern, e.g. `${BRAVE_API_KEY}` → `BRAVE_API_KEY=...` in env at launch]
- **`allowed-tools` usage:** [yes/no, with rationale visible from source]

## Tool design quality

- **`additionalProperties: false`** on input schemas? `<Yes / Partial / No>` (cite one example schema)
- **"Do NOT use for X"** anti-instructions in tool descriptions? `<Yes / Partial / No>`
- **Polymorphic shapes** (`anyOf`, discriminated unions)? `<Yes / No>` (cite if yes)

## Notable design decisions (lessons to copy)

- [Decision 1, with file:line]
- [Decision 2, with file:line]

## Anti-patterns observed (lessons NOT to copy)

- [If applicable, with file:line. If none, write "None observed in the audit."]

## Verification

To re-verify this annotation against current code:

\`\`\`bash
git clone <repo URL> /tmp/<slug>
cd /tmp/<slug>
git checkout <commit-sha>
\`\`\`

Then re-read the cited file:line locations.
```

Key discipline rules for the annotations:

- **No fabrication.** Every claim about behavior must cite a file:line in the source. If you can't find evidence, write "Not observed in the audit" — don't infer.
- **Quotes are short.** Verbatim source citations are fine but capped at ~5 lines per cite (copyright + readability).
- **Annotations are dated.** "Last analyzed" + "Commit" pin the audit to a frozen state. The README explicitly says "annotations may go stale; the dated-commit reference lets a reader re-verify."
- **No editorializing.** "This is bad" or "This is well-designed" isn't useful — describe the pattern and let the reader judge. The "lessons to copy / NOT to copy" sections frame it without value-loading every sentence.

### B.3 Phase B acceptance gates

- Each `mcps/*.md` file follows the exact template structure (CI markdownlint will catch some shape issues, but the template fields are content checks).
- Every claim cites a file:line.
- The aggregate has 10–15 MCPs (final count locked at plan-mode approval).
- Markdown link check (`lychee`) passes on the per-MCP files.

## Phase C — Synthesis

### C.1 Top-level README

`README.md` at repo root contains:

```markdown
# claude-mcp-patterns-audit

[![links](https://github.com/rkendev/claude-mcp-patterns-audit/actions/workflows/links.yml/badge.svg)](https://github.com/rkendev/claude-mcp-patterns-audit/actions/workflows/links.yml)
[![markdownlint](https://github.com/rkendev/claude-mcp-patterns-audit/actions/workflows/markdownlint.yml/badge.svg)](https://github.com/rkendev/claude-mcp-patterns-audit/actions/workflows/markdownlint.yml)

Curated audit of <N> production MCP servers — what canonical CCA-F D2
patterns ship in the wild, and what anti-patterns hide in plain sight.

Built as Artifact D of a Claude Certified Architect Foundations
small-projects portfolio. Companion to:

- [claude-mcp-server-minimal](https://github.com/rkendev/claude-mcp-server-minimal) (Artifact A — minimal MCP server)
- [claude-tools-roundtrip-playground](https://github.com/rkendev/claude-tools-roundtrip-playground) (Artifact B — native tools= round-trip)
- [claude-tool-choice-modes](https://github.com/rkendev/claude-tool-choice-modes) (Artifact C — tool_choice modes)

## How to use this audit

Pick an MCP from the table below and read its dedicated annotation file.
Each annotation cites file:line in the source — the dated commit
reference at the top lets you re-verify against the current code.

## Summary table

| MCP | Transport | Error envelope (D2) | `additionalProperties:false` | "Do NOT use for X" | Anti-patterns observed | File |
|---|---|---|---|---|---|---|
| <name> | stdio / SSE | Yes / Partial / No | Yes / No | Yes / No | <count> | [link](mcps/<slug>.md) |
| ... | ... | ... | ... | ... | ... | ... |

## Cross-cutting findings

[2–4 paragraphs, ≤300 words each. Examples of what to write here:]

- **What's universal:** Patterns that ALL audited MCPs share (e.g. "every audited stdio server reads env-vars at launch, none at first tool call").
- **What's contested:** Patterns where MCPs split (e.g. "5 of 12 use canonical errorCategory, 7 use raw exceptions").
- **The biggest anti-pattern in the wild:** Most common mistake observed.
- **What hiring conversations should test:** Concrete questions to ask a candidate based on what production MCPs actually do.

## Methodology

- Each MCP analyzed at a frozen commit (cited at the top of its annotation).
- Annotations are dated; re-verify against current code via the cited commit.
- Source observations are cited at file:line — no behavior is asserted without evidence.
- Anti-patterns are noted descriptively, not pejoratively. Real production code makes pragmatic trade-offs that look "wrong" out of context.

## License

MIT — see [LICENSE](LICENSE).
```

### C.2 CHANGELOG

```markdown
# Changelog

## [Unreleased]

## [0.1.0] — 2026-MM-DD

Initial audit. <N> MCP servers analyzed.

### Added

- `mcps/*.md` — per-MCP annotation files (<N> total).
- Top-level `README.md` with summary table + cross-cutting findings.
- `.github/workflows/links.yml` (lychee link checker).
- `.github/workflows/markdownlint.yml` (markdown style enforcement).

[Unreleased]: https://github.com/rkendev/claude-mcp-patterns-audit/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/rkendev/claude-mcp-patterns-audit/releases/tag/v0.1.0
```

(Don't tag `v0.1.0` in this PR. Wk8 ships the tag alongside Medium #3.)

### C.3 Phase C acceptance gates

- `README.md` exists with summary table populated for all <N> MCPs.
- Cross-cutting findings section has 2–4 paragraphs of substantive content (not placeholder).
- Both CI workflows green on the PR.

## Acceptance criteria (overall)

- 10–15 MCP annotation files in `mcps/`, each following the template exactly.
- Every behavioral claim cites a file:line.
- README summary table populated, no `<...>` placeholders left.
- Cross-cutting findings section has real paragraphs (no "TBD" or empty bullets).
- Both `links.yml` and `markdownlint.yml` workflows green on the PR.
- Public repo accessible at `https://github.com/rkendev/claude-mcp-patterns-audit`.
- After squash-merge, regression curl: `curl -sI https://raw.githubusercontent.com/rkendev/claude-mcp-patterns-audit/main/README.md | head -1` returns `HTTP/2 200`.

## Anti-patterns to avoid

- Do NOT write a Medium post. That's Wk8's task. This PR ships only the repo + annotations.
- Do NOT analyze MCPs that aren't actually deployed somewhere. Toy examples ≠ production patterns. The audit's value depends on the MCPs being real.
- Do NOT fabricate features. If you can't find error handling in an MCP's source, write "No error envelope observed" — don't invent one.
- Do NOT include long verbatim source quotes (>5 lines). Cite file:line, paraphrase, or use short snippets only. Copyright + readability.
- Do NOT use Python / `make check` / VCR / cassettes / `roy-ai-template` patterns. None apply here. The artifact is markdown.
- Do NOT add a `tests/` directory or unit tests. The "test" is the link checker passing and reviewers re-verifying file:line citations against the cited commits.
- Do NOT bump version, do NOT tag `v0.1.0`, do NOT touch a `Makefile`. Wk8 polish.

## Protocols (inlined)

- **Source verification:** Every behavioral claim cites a file:line in the original source. The `Last analyzed: <commit>` field at the top of each annotation pins the citation.
- **Markdown style:** `markdownlint-cli2` enforces. Don't disable rules in `.markdownlint.json` beyond the three already approved (line length, allowed HTML, first-line-h1).
- **Link discipline:** All external URLs must resolve. `lychee` will catch broken links on every push.
- **Branch protocol:** Single feature branch `feat/t005-mcp-audit`. PR-based; squash-merge with explicit approval (no auto-merge).
- **Commit cadence:** Phase A as one commit (scaffold). Phase B as one or several commits (one per MCP is fine; the squash-merge collapses them). Phase C as one commit (synthesis).
- **No Smoke note needed:** This repo has no `smoke.yml` — only `links.yml` and `markdownlint.yml`. Both run on PR + push.

## Plan mode requirements

Begin in plan mode. Walk through:

1. **The candidate list — primary deliverable of plan mode.** Propose 10–15 MCPs. For each:
   - Repo URL.
   - One-line rationale for inclusion (e.g. "Anthropic's reference filesystem MCP — canonical patterns, ~150 commits, 4k stars").
   - Estimated annotation depth (small / medium / large).
   - Pre-emptive flags: which anti-pattern category you expect to find, if any.
   The user will scrutinize this list before you write a single annotation.
2. The summary table column shape — confirm the seven columns proposed in §C.1 are right, or propose adjustments.
3. The cross-cutting-findings outline — propose what 2–4 paragraphs you expect to write once all annotations are done.
4. Phase B time estimate — given the candidate list, how many hours does it actually take? If the answer is >10 hrs, propose a smaller candidate list (8 MCPs is fine, the spec says 10–15).
5. PR body draft (in `tasks/pr_body.md`).
6. The full file list (created/modified) for all three phases.

Wait for explicit approval before any `mkdir`, `git init`, or annotation writing.

## PR body template (write to `tasks/pr_body.md`)

```markdown
## What

T005 — Artifact D (MCP Resources Audit). Curated audit of <N>
production MCP servers covering CCA-F D2 patterns and anti-patterns
observed in the wild. Closes Wks 8–9 of the CCA-F small-projects plan.

## Why

Closes Sev-≥3 D2 gaps in the briefing's Gap Register that no other
artifact addresses head-on:

- **MCP Resources / catalogs in the wild** — what shapes do production
  MCPs actually publish? The audit catalogues 10–15 examples.
- **Description-as-routing in production** — does the "Do NOT use for X"
  pattern from the CCA-F study notes actually appear in shipped code?
  The audit answers empirically.
- **Anti-patterns to spot in interview reviews** — concrete file:line
  examples of decisions that look right but break under load.

## How verified

- Both `links.yml` and `markdownlint.yml` workflows green on this PR.
- All <N> annotations follow the template (manual visual check;
  reviewers should also re-verify one citation by running the
  `Verification` block at the bottom of each annotation).
- Top-level README summary table populated for all <N> MCPs.
- Cross-cutting findings section has 2–4 paragraphs of substantive
  content.

## Out of scope

- Medium #3 ("MCP Patterns in the Wild") — Wk8 task, separate PR.
- `v0.1.0` tag — Wk8 polish.
- Live re-verification automation. Annotations are dated and cite
  commits; staleness is human-checkable, not CI-enforced.

## Closes

T005 of CCA-F small-projects plan v1.
```

End of PR body template.

## Out of scope (later in Wk8)

- Medium #3 publication.
- `v0.1.0` tag of this repo.
- Adding more MCPs beyond the 10–15 covered.
- A "live audit" GitHub Action that re-checks annotations against current code (interesting future work, but defer).

## Memory cross-references

This audit is the empirical lens for findings already in Cowork memory. When writing annotations, lean on these as the canonical CCA-F vocabulary:

- `feedback_tool_choice_relax_to_auto.md` — the relax-to-auto pattern. If any audited MCP demonstrates it, cite the parallel.
- `feedback_anthropic_sdk_vcr_replay.md` — testing pattern. If any audited MCP has VCR-style cassettes, note the precedent.
- The CCA-F study notes in `Claude-Cowork/Claude Certified Architect/` define `errorCategory` / `isRetryable` / `message` as the canonical D2 vocabulary. Don't re-derive.
