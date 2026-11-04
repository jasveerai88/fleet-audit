# Changelog

## v0.1.0 — 2026-11-04

First release.

- `audit FOLDER` walks a folder of Claude Code style agent
  definition files recursively and inventories every agent:
  name, description, description length, declared tools,
  model.
- Description collisions by token-set Jaccard similarity:
  WARN at 0.50 (`DESCRIPTION_COLLISION`), ERROR at 0.90
  (`DESCRIPTION_IDENTICAL`); missing or empty descriptions
  are ERRORs (`DESCRIPTION_MISSING`); descriptions under 40
  characters WARN (`DESCRIPTION_SHORT`, heuristic floor).
- Role-vs-grant mismatches (`ROLE_TOOL_MISMATCH`, WARN,
  labelled heuristic): read-intent agents carrying
  write-capable tools (Edit / Write / MultiEdit /
  NotebookEdit / Bash), and write-intent agents granted
  nothing that can modify a file. Agents declaring both
  intents are not flagged.
- Duplicate agent names are ERRORs (`DUPLICATE_NAME`), with
  drift between the definitions (description / tools /
  model) named in the finding.
- Files that are not agent definitions in the supported
  shape are ERRORs (`UNRECOGNIZED_FILE`) with the parse
  reason — never a crash, never a guess.
- Tool-grant visibility: no `tools` field is INFO
  (`TOOLS_UNSPECIFIED` — inherits everything); an
  explicitly empty list is WARN (`TOOLS_EMPTY`).
- A hand-written strict frontmatter parser for the flat
  scalar/list subset these files use (plain / quoted
  scalars, flow lists, block lists, comma-separated tools);
  anything outside the subset is a per-file error.
- Text and `--json` output (including `{"error": ...}` for
  usage and input errors); exit 0 / 1 / 2. Read-only —
  the test suite proves the audited folder's bytes are
  unchanged.
- Bundled fictional fixtures: a 10-file faulty fleet with
  every fault planted once (expected: 9 findings, 4 errors,
  4 warnings, 1 info) and a clean 3-agent fleet (expected:
  0 findings). 58 tests, all expectations hand-counted.
