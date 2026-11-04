# fleet-audit — Subagent Fleet Auditor

**Your agents are routed by their descriptions — so audit the descriptions, before runtime misroutes the work.**

When a tool like Claude Code delegates, it does not read your agents' system prompts. It matches the task against each agent's `description` and picks one. That makes the description the routing table — and routing tables rot: two agents end up describing the same job, a reviewer quietly keeps the Edit tool it was drafted with, the same name gets defined twice in different folders, and one agent has no description at all, so it can never be chosen on merit. A study of description-based routing (mer.vin) found a near-identical "twin" agent winning **0 of 25** routing contests against its sibling, a single verb flip rerouting **10 of 10** trials, and **27.3%** of tasks silently falling through to the general-purpose agent. AgenticKit's recent self-audit of its own 34-agent fleet found the same class of faults — overlapping descriptions and a reviewer carrying leftover write access.

fleet-audit is the static checker for that layer. Point it at a folder of agent definitions and it inventories every agent, computes pairwise description similarity, checks tool grants against the job each description promises, and catches duplicate names and unreadable files — **before** a live run misroutes. It is read-only, runs nothing, and needs nothing but Python's standard library.

## What it checks

| # | Check | Severity | What it catches |
|---|---|---|---|
| 1 | `DESCRIPTION_IDENTICAL` | ERROR | Two descriptions at token-set similarity **≥ 0.90** — routing between them is a coin flip |
| 2 | `DESCRIPTION_MISSING` | ERROR | An agent with no description (or an empty one) — it can never be routed to on merit |
| 3 | `DUPLICATE_NAME` | ERROR | The same agent name defined by two files — which one loads is scope-order luck; differing definitions are reported as drift |
| 4 | `UNRECOGNIZED_FILE` | ERROR | A `.md` file that is not an agent definition in the supported shape (no frontmatter, unclosed frontmatter, syntax outside the parsed subset…) — the runtime loads no agent from it either |
| 5 | `DESCRIPTION_COLLISION` | WARN | Two descriptions at similarity **≥ 0.50** — they plausibly claim the same tasks |
| 6 | `ROLE_TOOL_MISMATCH` | WARN | Grants vs promised job (heuristic, see below): a read-intent agent carrying write-capable tools (Edit / Write / MultiEdit / NotebookEdit / **Bash**), or a write-intent agent granted nothing that can modify a file |
| 7 | `DESCRIPTION_SHORT` | WARN | A description under **40 characters** (heuristic floor) — too little text to match on |
| 8 | `TOOLS_EMPTY` | WARN | An explicit empty tools list (`tools: []`) — the agent can use nothing at all; usually a placeholder that shipped |
| 9 | `TOOLS_UNSPECIFIED` | INFO | No `tools` field at all — the agent inherits every tool the runtime offers |

**The similarity metric, stated plainly:** the Jaccard index over each description's *set* of lowercase alphanumeric word tokens — `|A ∩ B| / |A ∪ B|`. Repeated words count once; there is no stopword removal and no stemming. It measures shared vocabulary, not meaning: two descriptions can collide on words while meaning different things, and two can mean the same thing in different words and score low. Treat a collision as "read these two side by side", not as a verdict.

**Role intent is a heuristic, and says so in every finding.** Intent is inferred from word stems in the name + description — read words: `review…`, `audit…`, `verif…`, `check…`, `inspect…`, "read-only"; write words: `build…`, `edit…`, `write…`, `writer`. An agent whose text declares *both* (a reviewer that also edits the changelog) is not flagged in either direction: the grant matches the promise. When a finding fires, the matched words are named in the message so you can judge the inference yourself.

## The supported file shape — stated precisely

v0.1.0 supports **Claude Code style agent files**: Markdown files that open with a `---` frontmatter block of flat `key: value` lines carrying `name` and `description`, with optional `tools` and `model`. Frontmatter values may be plain, single- or double-quoted scalars, inline flow lists (`tools: [Read, Edit]`), block lists (`- Read` lines), or a comma-separated scalar for `tools`. The folder is walked **recursively** (subfolders included; paths are shown relative to the folder you passed); every `.md` file in the tree is expected to be an agent file.

Anything outside that subset — nested maps, block scalars (`|` / `>`), duplicate keys, tabs, a missing `name` — is reported as `UNRECOGNIZED_FILE` with the reason, never guessed at and never a crash. Agent files in a *different* ecosystem's shape are likewise reported as unrecognized rather than misparsed. Other keys (`color`, `memory`, …) parse fine and are ignored.

## Usage

```console
$ python3 src/fleet_audit.py audit path/to/.claude/agents
$ python3 src/fleet_audit.py audit path/to/.claude/agents --json
```

Exits: **0** no errors · **1** any ERROR finding · **2** input errors (folder missing, not a folder, or no `.md` files found).

## The bundled fleets

`examples/faulty-fleet/` is a fictional 10-file fleet with every fault planted once: a near-identical description pair (similarity 0.963), two collision pairs (0.645, 0.625), a security reviewer still carrying Edit, the name `deploy-helper` defined twice with drifted definitions (the second copy lives in `extra/`, exercising the recursive walk), an agent with no description, one with no tools field, one with a 12-character description, and a plain notes file with no frontmatter. `examples/clean-fleet/` is a 3-agent fleet that passes everything (and exercises all three tools forms: flow list, block list, comma scalar).

Expected results, counted by hand from the rules and the token sets before the tool was run:

| Fleet | Files | Parsed | Findings | Errors | Warnings | Info | Exit |
|---|---|---|---|---|---|---|---|
| faulty-fleet | 10 | 9 | 9 | 4 | 4 | 1 | 1 |
| clean-fleet | 3 | 3 | 0 | 0 | 0 | 0 | 0 |

### Real run — the faulty fleet (verbatim)

```console
$ python3 src/fleet_audit.py audit examples/faulty-fleet
fleet-audit v0.1.0 — audit · folder faulty-fleet
10 agent files · 9 agents parsed · 1 unrecognized · walk: recursive (subfolders included)
Read-only: it edits nothing and runs nothing. It reads the agent definition files in the folder you point it at — every .md file in the tree is expected to be an agent file in the supported shape.
Similarity: the Jaccard index over each description's set of lowercase alphanumeric word tokens. Collision at 0.50, near-identical at 0.90; the short-description floor is 40 characters. Thresholds and intent words are heuristics — see SKILL.md.

INVENTORY (9):
  code-reviewer — 168 chars · tools: Read, Grep, Glob · model: sonnet · code-reviewer.md
  deploy-helper — 125 chars · tools: Bash, Read · model: (default) · deploy-helper.md
  docs-reviewer — 165 chars · tools: Read, Grep · model: sonnet · docs-reviewer.md
  deploy-helper — 128 chars · tools: Read, Write, Bash · model: sonnet · extra/deploy-helper.md
  general-researcher — 154 chars · tools: (not specified — inherits all) · model: haiku · general-researcher.md
  pr-reviewer — 178 chars · tools: Read, Grep · model: sonnet · pr-reviewer.md
  quick-fixer — 12 chars · tools: Edit, Read · model: (default) · quick-fixer.md
  security-reviewer — 171 chars · tools: Read, Grep, Edit · model: opus · security-reviewer.md
  silent-helper — 0 chars · tools: Read · model: (default) · silent-helper.md

ERRORS — agents that cannot be routed to, collide head-on, or cannot be read (4):
[DESCRIPTION_IDENTICAL] "code-reviewer" + "pr-reviewer" · code-reviewer.md, pr-reviewer.md
  the descriptions of "code-reviewer" and "pr-reviewer" are near-identical (token-set similarity 0.963, at or above the 0.90 line) — a router choosing between them has nothing to choose on, so delegation between them is a coin flip
  fix: differentiate the descriptions until each names work the other does not claim — or merge the two agents
[DESCRIPTION_MISSING] "silent-helper" · silent-helper.md
  agent "silent-helper" has no description — routing is description-matching, so this agent can only ever be reached by explicit name, never chosen on merit
  fix: write a description that says what the agent does and when to use it
[DUPLICATE_NAME] "deploy-helper" · deploy-helper.md, extra/deploy-helper.md
  the name "deploy-helper" is defined by 2 files (deploy-helper.md, extra/deploy-helper.md) — which definition loads depends on scope order, so the fleet does not reliably contain the agent you think it does; and the definitions disagree on description, tools, model — the scopes have drifted apart
  fix: keep one definition per name in the fleet, or rename until names are unique
[UNRECOGNIZED_FILE] onboarding-notes.md
  this file cannot be read as an agent definition in the supported shape: no frontmatter block (the file does not open with a --- line) — the runtime loads no agent from it either
  fix: give the file frontmatter in the supported shape (--- name / description / tools / model ---), or move it out of the agents folder

WARNINGS — collisions and mismatches that will misroute work (4):
[DESCRIPTION_COLLISION] "code-reviewer" + "docs-reviewer" · code-reviewer.md, docs-reviewer.md
  the descriptions of "code-reviewer" and "docs-reviewer" overlap heavily (token-set similarity 0.645, at or above the 0.50 line) — they plausibly claim the same tasks, so routing between them will be inconsistent
  fix: sharpen the boundary: name the tasks each one owns, and in each description say what the other is for
[DESCRIPTION_COLLISION] "docs-reviewer" + "pr-reviewer" · docs-reviewer.md, pr-reviewer.md
  the descriptions of "docs-reviewer" and "pr-reviewer" overlap heavily (token-set similarity 0.625, at or above the 0.50 line) — they plausibly claim the same tasks, so routing between them will be inconsistent
  fix: sharpen the boundary: name the tasks each one owns, and in each description say what the other is for
[DESCRIPTION_SHORT] "quick-fixer" · quick-fixer.md
  agent "quick-fixer" has a 12-character description — under the 40-character floor (heuristic): too little text for a router to match on
  fix: expand the description: what it does, what it works on, when to use it
[ROLE_TOOL_MISMATCH] "security-reviewer" · security-reviewer.md
  agent "security-reviewer" reads as read-only work (heuristic: intent inferred from reviewer, audits, checks, audit in its name / description) but its tools grant write access: Edit — a reviewer that can edit will eventually produce diffs instead of verdicts
  fix: drop the write tools — or, if it really should write, change the description so the intent is honest

INFO — worth knowing (1):
[TOOLS_UNSPECIFIED] "general-researcher" · general-researcher.md
  agent "general-researcher" declares no tools field — it inherits every tool the runtime offers, the widest grant there is
  fix: declare the tools it actually needs; an explicit list is the fleet's permission boundary

Summary: 9 findings · 4 errors · 4 warnings · 1 info
Verdict: FAIL — 4 error(s): the fleet has agents that cannot be routed to, collide head-on, or cannot be read at all. Fix the errors before trusting delegation to this fleet.
(exit 1)
```

### Real run — the clean fleet (verbatim)

```console
$ python3 src/fleet_audit.py audit examples/clean-fleet
fleet-audit v0.1.0 — audit · folder clean-fleet
3 agent files · 3 agents parsed · 0 unrecognized · walk: recursive (subfolders included)
Read-only: it edits nothing and runs nothing. It reads the agent definition files in the folder you point it at — every .md file in the tree is expected to be an agent file in the supported shape.
Similarity: the Jaccard index over each description's set of lowercase alphanumeric word tokens. Collision at 0.50, near-identical at 0.90; the short-description floor is 40 characters. Thresholds and intent words are heuristics — see SKILL.md.

INVENTORY (3):
  api-builder — 130 chars · tools: Read, Edit, Write, Bash · model: sonnet · api-builder.md
  release-notes-writer — 123 chars · tools: Read, Write · model: haiku · release-notes-writer.md
  test-reviewer — 137 chars · tools: Read, Grep · model: sonnet · test-reviewer.md

No findings — every agent passes the static rules.
Summary: 0 findings · 0 errors · 0 warnings · 0 info
Verdict: CLEAN — no collisions, mismatches or unreadable files found by the static rules.
(exit 0)
```

### For agents and scripts — `--json`

One JSON object on stdout, errors included (exit 2 shown last):

```console
$ python3 src/fleet_audit.py audit examples/faulty-fleet --json
{
  "tool": "fleet-audit",
  "version": "0.1.0",
  "command": "audit",
  "folder": "faulty-fleet",
  "files": 10,
  "agents_parsed": 9,
  "unrecognized": 1,
  "thresholds": {
    "collision": 0.5,
    "identical": 0.9,
    "short_description_chars": 40
  },
  "agents": [
    {
      "name": "code-reviewer",
      "file": "code-reviewer.md",
      "description": "Reviews code changes for bugs, regressions, and style problems before merge. Use when a pull request or diff needs a careful read-only review of implementation quality.",
      "description_chars": 168,
      "tools": [
        "Read",
        "Grep",
        "Glob"
      ],
      "model": "sonnet"
    },
    … (8 more agents, then "findings": the same 9 findings with rule / severity / agents / files / similarity / excerpts / message / fix each) …
  ],
  "summary": {
    "findings": 9,
    "errors": 4,
    "warnings": 4,
    "info": 1
  },
  "clean": false
}
(exit 1)

$ python3 src/fleet_audit.py audit /no/such/folder --json
{"error": "folder not found: /no/such/folder"}
(exit 2)
```

`SKILL.md` carries the full JSON schema, the complete rule table and the honesty rules for agents driving this tool.

## Tests

58 tests — every check in both directions, the similarity metric pinned on hand-counted token sets (including exactly-at-the-line pairs: 0.50 warns, 0.90 errors, 0.80 only warns), the frontmatter parser's edges (flow/block/comma tools forms, quoted scalars with colons, CRLF, nested maps and block scalars reported unrecognized, duplicate keys, missing names), the JSON summary recounted against the findings list, and a read-only proof that hashes every fixture file before and after a run:

```console
$ python3 -m unittest discover -s tests
Ran 58 tests in 3.319s

OK
```

## Who it's for

Anyone running a fleet of subagents — a `.claude/agents/` folder that has grown past the point where one person remembers every description. Run it after adding or editing agents, in CI on the agents folder, or before trusting a downloaded agent pack. Inside our own engine it audits the 200-plus role fleet Atlas routes work through, where exactly these faults — twins, drifted duplicates, reviewers with write tools — are the expensive kind.

## Known limits

- **Static only.** A clean audit means the *files* don't collide; it does not prove live routing is correct — model behaviour, task wording and runtime fallthrough are outside what a static check can see. A live routing benchmark is a different tool.
- **Vocabulary, not meaning.** The token-set metric misses paraphrase collisions (same job, different words) and over-fires on boilerplate-heavy descriptions. Read flagged pairs; don't auto-edit from the number.
- **One file shape.** Claude Code style frontmatter only in v0.1.0; other ecosystems' agent files are reported unrecognized, not parsed. The strict parser covers the flat scalar/list subset those files actually use and rejects the rest by name.
- **Intent is inferred, narrowly.** The ROLE_TOOL_MISMATCH word lists are deliberately small to avoid false flags on ordinary prose ("implementation", "writing quality"); an agent that promises review work without any review-flavoured word will not be intent-checked.
- **`tools:` with an empty value is read as "field absent"** (inherit-all, INFO) — the two spellings are indistinguishable once written; an explicit `tools: []` is the WARN instead.

## Privacy

Everything is local. The auditor reads only the folder you point it at, writes nothing, and contains no network code at all — no telemetry, no uploads, no accounts. The findings describe your files; they never leave your machine.

## Support

fleet-audit is free and MIT-licensed. If it saves you a misrouted afternoon, sponsoring keeps tools like it coming: **https://github.com/sponsors/jasveerai88**

## Licence

MIT — see `LICENSE`.
