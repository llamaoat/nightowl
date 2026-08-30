# nightowl

**Picks which of your stalled Claude Code sessions is worth finishing, and can finish it.**

Claude Code already schedules work — `/schedule`, Desktop scheduled tasks, and
cloud Routines all run a prompt on a cadence. What none of them do is decide
*which* of your unfinished sessions deserves the slot. That's this.

```
  nightowl  ·  unused quota, unfinished work  ·  last 2w

  5h window   ████████░░░░░░░░░░░░░░░░  62k left of 190k  · resets in 38m

  ■ 15k spendable (after 25% block + 20% weekly reserve)

  ▸ 1. Add unit tests for the parser
      score 87  ·  ~/dev/invoice-parser (main)
      needs ~30k  ·  3h ago
      why: 3/5 todos done · stopped at a usage limit · remaining work is mechanical
      resume: nightowl run aaaa1111
```

Reading your history costs zero tokens — local file parsing and `git`, no model
calls. No dependencies, no telemetry.

---

## Install

Node 18+ and Claude Code.

```bash
git clone https://github.com/llamaoat/nightowl.git
cd nightowl && npm link
nightowl
```

Reads `~/.claude/projects`, prints the dashboard, changes nothing.

As a Claude Code plugin (adds `/nightowl` and budget reminders during runs):

```
/plugin marketplace add llamaoat/nightowl
/plugin install nightowl@nightowl-marketplace
```

---

## Use

```bash
nightowl --window 1w                   # dashboard: 5d, 1w, 2w, 30d, all
nightowl --window 1w --show-excluded   # what it filtered out, and why
nightowl explain aaaa1111              # the scoring arithmetic
nightowl run                           # dry run — prints the prompt, spends nothing
nightowl run --go                      # actually run it
nightowl run --go --allow-edits        # permit file edits, in an isolated worktree
```

`--go` is required to spend anything. Runs are read-only unless `--allow-edits`.

Check `--show-excluded` early. It drops plain chats, read-only research, and
finished work — confirm it isn't dropping something you care about.

---

## How it ranks

```
score = finishability × autonomy × fit × 100
```

Multiplied, so a zero on any axis disqualifies.

**finishability** — how close to done, from the session's todo list. Peaks
around 80%: far enough to plausibly finish, not so far that nothing's left.
×1.6 if it stopped at a usage limit and was never picked back up, ×0.25 if the
last message reads like a sign-off. Decays with a 3-day half-life.

**autonomy** — can it proceed without you? A session parked on *"Redis or
Postgres?"* scores ×0.15 regardless of how close to done it is.

**fit** — does the estimated cost fit the budget? Estimated from the session's
own observed tokens-per-todo, not a constant.

`nightowl explain <id>` prints the arithmetic for any candidate.

Before scoring, it asks git what happened *after* the session ended: remaining
todos appearing in later commits (`superseded`, ×0.05), a dormant repo, a
deleted branch, uncommitted work still in the tree.

---

## Correcting it

The completeness figure comes from Claude's todo list, which drifts from
reality.

```bash
nightowl progress aaaa1111 80%      # or 0.8, or 4/5
nightowl progress aaaa1111 --clear
nightowl verdict aaaa1111 dead      # never suggest again
nightowl verdict aaaa1111 live      # still working on it
```

Your figure replaces the todo ratio but not the curve — claiming 100% *lowers*
the score, since a finished session has nothing left to do. To run something
regardless of ranking, name it: `nightowl run <id> --go`.

Overrides expire if you work in that session again.

---

## Measuring it

```bash
nightowl eval          # does the ranking beat "just pick the most recent"?
nightowl outcomes      # did you keep the work? (from worktree fate)
nightowl calibrate     # were cost estimates right? --apply to correct them
```

`eval` replays your history, reconstructs what nightowl would have seen at each
past moment, and checks its pick against what you actually resumed. It compares
against naive baselines and says plainly when it loses.

Three things keep it honest: git queries are bounded to what existed at the
replay point (no hindsight), sessions too recent to have a label are excluded
rather than counted as negatives, and every figure carries a Wilson interval.

The label is "did you resume it", not "was it worth resuming" — read the
numbers as an upper bound on badness.

---

## During a run

nightowl reopens the session and builds the prompt from its own leftover todo
list. Budget reminders fire at 50%, 75% and 90% spent, each narrowing scope
rather than starting new work. At 90% it writes a handoff note and stops.

Notifications on finish, budget exhaustion, or failed launch:

```bash
nightowl notify --test
```

Desktop needs no setup. For your phone, add an [ntfy](https://ntfy.sh) topic to
`~/.nightowl/config.json`:

```jsonc
{ "notify": { "ntfy": "https://ntfy.sh/your-unique-topic" } }
```

Pick something unguessable — anyone with the topic can read it.

---

## Scheduling

Use Claude Code's own: Desktop scheduled tasks, or cloud Routines, which run
with your laptop closed. nightowl picks the target; they run it.

```bash
nightowl status --json | jq -r '.candidates[0].sessionId'
```

`nightowl watch --go` polls locally and fires when quota is about to expire. It
needs a terminal left open and predates Routines. Kept for now; use Routines
instead.

---

## Safety

On by default, off only deliberately:

- nothing spent without `--go`
- no edits without `--allow-edits`
- edits happen in a detached git worktree, never your branch
- 25% of the 5-hour window and 20% of the week are never spendable
- a token ceiling, enforced by killing the run
- 45-minute timeout
- no commits, pushes or tags
- below score 25 it refuses to pick

`--dangerously-skip-permissions` is unreachable from every code path.

The ceiling is checked after each turn, so a run can overshoot by up to one
turn's cost. On a small budget that overshoot is proportionally large.

---

## Config

Optional, `~/.nightowl/config.json`. See `config.example.json`.

```jsonc
{
  "window": "2w",
  "reserve": { "blockPct": 0.25, "weeklyPct": 0.20 },
  "run": { "allowEdits": false, "useWorktree": true, "timeoutMinutes": 45 },
  "ignore": ["/work/client-repos"]
}
```

`NIGHTOWL_ASK=off` silences the occasional question, `NIGHTOWL_NUDGE=off` the
session-start nudge.

---

## Limitations

**Quota figures are estimates.** No public API reports a subscription's
remaining tokens. Everything is inferred from your transcripts and calibrated
against your history. Good for "do I have room and is it expiring"; not a
billing oracle.

**Claude Code only.** claude.ai chat history isn't readable from your machine —
no local store, no read API. A Gemini CLI adapter exists on the `adapters`
branch, written from docs and unverified.

**Overlaps with shipped features.** Claude Code auto-continues limit-interrupted
sessions by default, so nightowl skips recent limit-stops and focuses on older
and non-limit stalls. Desktop scheduled tasks also offer worktree isolation.

**The transcript format is undocumented and moves between versions.** nightowl
parses defensively and handles both known directory layouts. If an update
breaks it, the symptom is "no stalled sessions found" when you know you have
some — that's a bug report, not an empty history.

**The ranking is tuned on synthetic fixtures, not real data.** `nightowl eval`
exists to find out whether it's wrong for you.

**POSIX only.** Linux and macOS, Node 18/20/22.

---

## Contributing

Most useful contribution: `nightowl eval` output from real history, especially
when nightowl ties or loses to a baseline. After that, a failing fixture for a
bad ranking.

[CONTRIBUTING.md](CONTRIBUTING.md) · [DESIGN.md](DESIGN.md) for why things work
the way they do.

MIT
