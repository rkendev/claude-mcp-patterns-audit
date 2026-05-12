# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] — 2026-05-12

First stable release of `claude-mcp-patterns-audit`: a curated audit of 13
production MCP server implementations, cataloguing architectural choices and
anti-patterns at frozen commits, with cross-cutting findings on error
envelopes, schema strictness, and "Do NOT use for X" anti-instructions.

### Added

- **T005 — Initial 13-MCP audit** (PR #1, squash-merged commit `8b46539`).
  Per-server annotations under `mcps/*.md` covering Architecture, Error
  handling, Configuration, Tool design quality, Notable design decisions,
  and Anti-patterns observed. Top-level `README.md` with summary table
  (13 rows) and cross-cutting findings on the audit's biggest pattern in
  the wild: lossy error envelopes across 12 of 13 servers.

- **T008 — Audit content fix** (this PR). One non-blocking correction
  surfaced during the T005 self-audit pass:
  - Summary-table `x-twitter-mcp-server` cell: `5+` → `9` (matches the
    nine bullets enumerated under `mcps/x-twitter-mcp-server.md`'s
    `## Anti-patterns observed` section).

  Total: 14 markdown files (1 top-level + 13 under `mcps/`), all outbound
  links checked by `links.yml` (lychee) and style enforced by
  `markdownlint.yml`.

[Unreleased]: https://github.com/rkendev/claude-mcp-patterns-audit/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/rkendev/claude-mcp-patterns-audit/releases/tag/v0.1.0
