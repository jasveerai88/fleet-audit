---
name: fleet-audit
description: Statically audit a folder of Claude Code style subagent definition files before runtime routes work through them. Inventories every agent (name, description length, tools, model) and reports description collisions (token-set Jaccard similarity — WARN at 0.50, ERROR at 0.90), missing descriptions, duplicate agent names with drift, unreadable files, tool grants that mismatch the promised role (heuristic intent inference), and agents that inherit all tools. Read-only, offline, stdlib only. Use it after adding or editing agents, in CI on an agents folder, or before trusting a downloaded agent pack. NOT for live routing benchmarks, editing agent files, or any other ecosystem's agent-file shape.
---

# fleet-audit — agent skill

## Purpose

When a runtime delegates to subagents, it routes by matching the
task against each agent's `description`. fleet-audit checks that
routing layer statically: it reads a folder of agent definition
files and reports the collisions, mismatches and unreadable
files that would otherwise surface as misrouted work in a live
run.

The honesty core:

- Findings are about the FILES. A clean audit means no static
  collisions were found — it is NOT proof that live routing is
  correct, and the tool never claims it is.
- Similarity is shared vocabulary, not meaning. The metric is
  stated below so a reader can judge any flagged pair.
- Role intent is INFERRED from keywords and labelled a
  heuristic in every finding that uses it — never present an
  intent finding as a fact about the fleet. The finding names
  the matched words so a human can overrule it.
- Never edit agent files to silence a finding without reading
  the pair (collision / identical) or the agent (mismatch)
  first. The fix text is a starting point, not an order.

## When to use

- After adding, renaming or editing agents in a fleet folder.
- In CI, gating changes to a `.claude/agents/` directory.
- Before installing or trusting a third-party agent pack.
- When delegation "feels random" — check for twins and
  description-less agents before blaming the model.

## When NOT to use

- To measure live routing accuracy (that needs a benchmark
  that actually delegates tasks; this tool is static).
- On agent files in another ecosystem's shape — they will be
  reported UNRECOGNIZED_FILE, which is the honest answer, not
  a parse.
- To auto-rewrite descriptions. Differentiation is a design
  decision; the tool only locates the collisions.

## Supported input shape (v0.1.0)

Claude Code style agent files: Markdown opening with a `---`
frontmatter block of flat `key: value` lines; `name` and
`description` carried, `tools` / `model` optional. Values:
plain, single- or double-quoted scalars; inline flow lists
`[a, b]`; block lists (`- item` lines); `tools` may also be a
comma-separated scalar. Other keys parse and are ignored.
Anything else — nested maps, block scalars (`|` / `>`),
duplicate keys, tabs in frontmatter, no frontmatter, no
`name` — is an UNRECOGNIZED_FILE finding with the reason, per
file, never a crash. The folder is walked RECURSIVELY;
subfolders are included and paths are shown relative to the
folder passed. Every `.md` file in the tree is expected to be
an agent file.

## Commands

```console
python3 src/fleet_audit.py audit FOLDER           # grouped text report
python3 src/fleet_audit.py audit FOLDER --json    # one JSON object
```

## Rules (complete)

Similarity: **Jaccard index over token sets** —
tokens = lowercase alphanumeric runs (`[a-z0-9]+`) of the
description; similarity = |A ∩ B| / |A ∪ B|. Repeated words
count once; no stopword removal, no stemming. Each pair earns
at most one pair finding (identical beats collision). Agents
without descriptions are skipped by pair checks.

| Rule | Severity | Trigger |
|---|---|---|
| `DESCRIPTION_IDENTICAL` | ERROR | pair similarity ≥ 0.90 |
| `DESCRIPTION_MISSING` | ERROR | `description` absent or whitespace-only |
| `DUPLICATE_NAME` | ERROR | same `name` in 2+ files; message lists which of description / tools / model differ (drift) |
| `UNRECOGNIZED_FILE` | ERROR | file not readable as an agent in the supported shape; reason given |
| `DESCRIPTION_COLLISION` | WARN | 0.50 ≤ pair similarity < 0.90 |
| `ROLE_TOOL_MISMATCH` | WARN | intent vs grants (below) |
| `DESCRIPTION_SHORT` | WARN | description < 40 characters (heuristic floor) |
| `TOOLS_EMPTY` | WARN | `tools` present but an explicitly empty list |
| `TOOLS_UNSPECIFIED` | INFO | no `tools` field — inherits every tool |

Intent inference (heuristic): searched over `name` +
description. Read words — stems `review`, `audit`, `verif`,
`check`, `inspect` (any word starting with them) and the
phrase "read-only". Write words — `build`, `builds`,
`builder`, `building`, `edit`, `edits`, `editor`, `editing`,
`write`, `writes`, `writer`. Write-capable tools: `Edit`,
`Write`, `MultiEdit`, `NotebookEdit`, and `Bash` (it can
modify files). Flagged: read-intent-only + any write-capable
grant; write-intent-only + an explicit tool list granting
nothing write-capable. If BOTH intents appear, no flag either
way — the grant matches a promise that includes writing.
Tool names are matched case-insensitively for this check and
reported in canonical spelling.

Field notes: `tools:` with an empty value is treated as the
field being absent (INFO); `tools: []` is TOOLS_EMPTY (WARN).

## JSON schema

```json
{
  "tool": "fleet-audit", "version": "0.1.0",
  "command": "audit",
  "folder": "<folder name only>",
  "files": 10, "agents_parsed": 9, "unrecognized": 1,
  "thresholds": {"collision": 0.5, "identical": 0.9,
                 "short_description_chars": 40},
  "agents": [{"name": "...", "file": "rel/path.md",
              "description": "..." | null,
              "description_chars": 0,
              "tools": ["Read", ...] | null,
              "model": "sonnet" | null}],
  "findings": [{"rule": "DESCRIPTION_COLLISION",
                "severity": "ERROR" | "WARN" | "INFO",
                "agents": ["a", "b"], "files": ["a.md"],
                "similarity": 0.645 | null,
                "excerpts": {"a": "..."} | null,
                "message": "...", "fix": "..."}],
  "summary": {"findings": 9, "errors": 4, "warnings": 4,
              "info": 1},
  "clean": false
}
```

Notes: `tools: null` in the inventory means the field is
absent (inherit-all); `tools: []` means explicitly empty.
`clean` is true exactly when `errors` is 0. Findings are
sorted deterministically (severity, rule, agents, files).
Errors (including argparse errors under `--json`) print
`{"error": "..."}` and exit 2.

## Exit codes

- `0` — no ERROR findings
- `1` — at least one ERROR finding
- `2` — input error (folder missing / not a folder / no
  `.md` files found) or a usage error

## Limits & honesty rules

- Static analysis only: never report a fleet as "routing
  correctly" — only as free of the static faults above.
- The thresholds (0.50 / 0.90 / 40 chars) are conventions,
  stated so they can be argued with; they are not measured
  constants of any runtime.
- Paraphrase collisions (same job, different words) score
  LOW and are missed by design of the metric; boilerplate-
  heavy descriptions score HIGH and may be false alarms.
  Both directions are why findings say "read this pair".
- Intent word lists are deliberately narrow (broader stems
  false-positive on ordinary reviewer prose); an agent whose
  description avoids every intent word is not intent-checked.
- Read-only: the auditor never writes to the audited folder
  (the test suite proves fixture bytes unchanged).
- Two checks were considered and DELIBERATELY left out:
  descriptions lacking a negative boundary ("do not use
  for…") — most real fleets would flag, turning INFO into
  noise; and filename-vs-`name` mismatch — the `name` field
  is the routed identity, so flagging the filename would be
  a false certainty.
- Fixtures under `examples/` are fictional fleets built to
  plant each fault once; their expected findings were counted
  by hand from this rule table, not from the tool's output.
