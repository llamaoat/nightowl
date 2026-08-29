# Contributing

## The useful contributions

The ranking heuristics in `lib/score.mjs` are the interesting surface and the
weakest part — they were tuned against five synthetic fixtures, not a corpus.
If `nightowl` ranks something obviously wrong for you, **a fixture reproducing
it is worth more than a heuristic tweak.** Add it to `test/make-fixtures.mjs`
and assert the correct ordering in `test/run-tests.mjs`.

The budget tracker's phase thresholds (50/75/90) and wording are informed
guesses ported from published results on other agent scaffolds. They have never
been A/B'd on Claude Code. `~/.nightowl/runs.jsonl` logs `budgetTokens`,
`spent`, `killedForBudget`, and injection counts for every run. If you gather
that data across real tasks, a PR with numbers beats any amount of tuning.

## The most valuable PR

Run `nightowl eval` on your real history and open an issue with the output —
**especially if the model ties or loses to a baseline.** That is the single
most useful contribution to this project, more than any heuristic tweak.

If enough people do that, the next step is a shared corpus. `anonymiseRow()` in
`lib/eval.mjs` emits feature vectors and labels only — counts, ages, enum
verdicts, no paths, no file names, no prompt text, no commit messages. The plan
is voluntary PRs of those vectors, never telemetry. This project does not phone
home and should not start.

## Ground rules

- **No dependencies.** This runs unattended with access to people's repos and
  quota. The supply chain surface stays at zero.
- **Safety defaults never loosen.** Dry-run, read-only, worktree isolation, and
  the reserves are opt-out only. `bypassPermissions` must remain unreachable
  from every code path.
- **Hooks must fail silently.** `hooks/budget.mjs` runs after every tool call in
  every session on the machine. It must never exit non-zero, never write to
  stderr on the happy path, and never add measurable latency when idle.
- **Keep `npm test` green.** The suite asserts behavioural invariants, not exact
  scores — scores may drift, orderings may not.

## Running things

```bash
npm run fixtures                    # write synthetic transcripts to /tmp
npm test                            # ranking + tracker suites
NIGHTOWL_CONFIG=/tmp/c.json nightowl status --json
```

`NIGHTOWL_STATE_DIR` and `NIGHTOWL_CONFIG` redirect all state, so tests never
touch your real `~/.nightowl`.

## Platform

POSIX only for now (Linux and macOS, tested in CI). Windows support needs the
path handling in `lib/sessions.mjs` and the shell guard in `hooks/hooks.json`
rewritten; PRs welcome.
