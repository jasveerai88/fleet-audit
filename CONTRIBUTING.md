# Contributing to fleet-audit

Thanks for considering a contribution. fleet-audit has a
small, sharp job — statically auditing agent definition
files — and contributions are judged against keeping that
job honest.

## Ground rules

- **Pure Python standard library only.** No dependencies,
  no network code, no telemetry. If a change needs a
  package, it doesn't belong here.
- **Read-only, always.** The auditor must never write to
  the folder it audits (or run anything in it). The test
  suite hashes fixture bytes before and after a run; keep
  that proof green.
- **The rule table is the product.** Every check lives in
  the table in SKILL.md with its severity and basis. A new
  check needs a row there, a README mention, and tests in
  both directions (fault present earns the finding; fault
  absent earns none).
- **No false certainties.** Thresholds and intent words are
  heuristics and stay labelled as such, in the docs and in
  the findings themselves. Never let output wording claim a
  fleet "routes correctly" — only that no static faults
  were found.
- **Intent stays narrow.** The ROLE_TOOL_MISMATCH word
  lists are small on purpose: broader stems false-positive
  on ordinary prose. A proposal to widen them needs fixture
  evidence of a missed real case AND proof of no new false
  flags on the bundled fleets.
- **The parser rejects, it never guesses.** Frontmatter
  outside the documented flat scalar/list subset is an
  UNRECOGNIZED_FILE with a named reason. Do not loosen the
  parser to "probably parse" more shapes; a wrong parse is
  worse than an honest rejection.
- **Fixture expectations are counted by hand.** The
  expected findings for `examples/` were derived from the
  rule table and the token sets — if a change alters them,
  re-derive them by hand in the test docstring and say so
  in the PR. Never paste the tool's own output in as the
  expectation.

## Development

```console
python3 -m unittest discover -s tests -v
```

Python 3.10+. Please keep the style of the existing code:
small functions, no cleverness, messages written for the
person whose fleet just failed a check.

## Reporting issues

Open an issue with the smallest agent file (or pair) that
reproduces the problem, the command you ran, and the output
you got vs the output you expected. For a similarity
dispute, include the two descriptions and your own token
count — the metric is simple enough to argue about
precisely, and that is by design.
