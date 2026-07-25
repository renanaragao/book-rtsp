# Copilot instructions for `book-rtsp` (Markdown book)

## Build, test, and lint commands

This repository is for a book written in Markdown. It is currently uninitialized (no commits and no tracked project files), so there are no defined build, test, or lint commands yet.

- **Full suite commands:** not available yet
- **Single-test command pattern:** not available yet

## High-level architecture

The expected architecture is documentation-first rather than application code:

- Markdown source files are the primary artifact (chapters/sections).
- Cross-file structure matters (chapter ordering, internal links, and references).
- Tooling, if added later, should support documentation quality (formatting, link/reference validation, optional publish/export pipeline).

## Key conventions in this repository

- Optimize edits for long-form Markdown content consistency across files, not code compilation.
- Preserve link integrity when renaming/moving files (update all inbound relative links and references in the same change).
- Keep command guidance tied to actual repo tooling only; if markdown tooling is introduced (for example via `package.json`/`Makefile`), update this file with exact full-suite and single-file validation commands.

## MCP servers

- MCP configuration lives in `.vscode/mcp.json`.
- Use the `github` MCP server to support repository-centric workflows for this book project (issues, PR context, and repository file operations).

<!-- rtk-instructions v2 -->
# RTK — Token-Optimized CLI

**rtk** is a CLI proxy that filters and compresses command outputs, saving 60-90% tokens.

## Rule

Always prefix shell commands with `rtk`:

```bash
# Instead of:              Use:
git status                 rtk git status
git log -10                rtk git log -10
cargo test                 rtk cargo test
docker ps                  rtk docker ps
kubectl get pods           rtk kubectl pods
```

## Meta commands (use directly)

```bash
rtk gain              # Token savings dashboard
rtk gain --history    # Per-command savings history
rtk discover          # Find missed rtk opportunities
rtk proxy <cmd>       # Run raw (no filtering) but track usage
```
<!-- /rtk-instructions -->