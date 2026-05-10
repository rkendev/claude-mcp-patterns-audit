## What

T005 — Artifact D (MCP Resources Audit). Curated audit of 13
production MCP servers covering CCA-F D2 patterns and anti-patterns
observed in the wild. Closes Wks 8–9 of the CCA-F small-projects plan.

## Why

Closes Sev-≥3 D2 gaps in the briefing's Gap Register that no other
artifact addresses head-on:

- **MCP Resources / catalogs in the wild** — what shapes do production
  MCPs actually publish? The audit catalogues 13 examples.
- **Description-as-routing in production** — does the "Do NOT use for X"
  pattern from the CCA-F study notes actually appear in shipped code?
  The audit answers empirically.
- **Anti-patterns to spot in interview reviews** — concrete file:line
  examples of decisions that look right but break under load.

## How verified

- Both `links.yml` and `markdownlint.yml` workflows green on this PR.
- All 13 annotations follow the template (manual visual check;
  reviewers should also re-verify one citation by running the
  `Verification` block at the bottom of each annotation).
- Top-level README summary table populated for all 13 MCPs.
- Cross-cutting findings section has 2–4 paragraphs of substantive
  content.

## Out of scope

- Medium #3 ("MCP Patterns in the Wild") — Wk8 task, separate PR.
- `v0.1.0` tag — Wk8 polish.
- Live re-verification automation. Annotations are dated and cite
  commits; staleness is human-checkable, not CI-enforced.

## Closes

T005 of CCA-F small-projects plan v1.
