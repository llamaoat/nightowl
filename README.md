# nightowl

**You have forty stalled Claude Code sessions. Which one deserves the next hour?**

nightowl reads your local Claude Code history, works out which unfinished
sessions are actually close to done, and — if you want — spends surplus quota
finishing one while you're not there.

```
  nightowl  ·  unused quota, unfinished work  ·  last 2w

  5h window   ████████░░░░░░░░░░░░░░░░  62k left of 190k  · resets in 38m
  weekly      ██████████████████░░░░░░  4.1M left of 6.0M

  ■ 15k spendable (after 25% block + 20% weekly reserve)
  ▲ Use it or lose it — 62k expires in 38m

  ▸ 1. Add unit tests for the parser
      score 87  ·  ~/dev/invoice-parser (main)
      needs ~30k  ·  3h ago
      why: 3/5 todos done · stopped at a usage limit · remaining work is mechanical
      resume: nightowl run aaaa1111
```

No dependencies. No telemetry. **Reading your history costs zero tokens** — it
is local file parsing and `git`, not model calls.

---

## Try it in 30 seconds

Requires Node 18+ and Claude Code installed.

```bash
git clone https://github.com/llamaoat/nightowl.git
cd nightowl && npm link
nightowl
```

That's it. It reads `~/.claude/projects`, prints the dashboard, and **changes
nothing**. Nothing written, nothing spent, no files touched.

If it says *"No Claude Code sessions found"*, you haven't used Claude Code on
this machine yet — that's the only requirement.

---

## Your first five minutes

### 1. See what it found

```bash
nightowl --window 1w
```

One week is a good place to start. `2w` is the default; `5d`, `30d` and `all`
also work.

### 2. See what it ignored

```bash
nightowl --window 1w --show-excluded
```

This is the important one. nightowl throws out plain chats ("explain this
error"), read-only research, and finished work. **Check it didn't bin something
you care about.** If it did, that's worth reporting — the known blind spot is a
design discussion with no file edits and no todo list.

### 3. Ask why something is ranked where it is

```bash
nightowl explain aaaa1111        # first few characters of the session id
```

You get arithmetic, not a vibe:

```
  score 87 = finishability 0.87 × autonomy 1.00 × fit 1.00 × 100
  signals
    · 3/5 todos done
    · stopped at a usage limit and never picked back up
    · remaining work is mechanical
  cost estimate ~30k via per-completed-todo
```

If you disagree, you can point at the number that's wrong.

### 4. Look at what it *would* do

```bash
nightowl run
```

**This spends nothing.** It prints the exact instructions it would send, then
stops. Read them. This is the moment to decide whether you trust it.

---

## Actually running something

```bash
nightowl run --go
```

`--go` is required. Without it, everything is a dry run, always.

By default a run is **read-only** — it can look at your code and think, but not
change it. To let it edit files:

```bash
nightowl run --go --allow-edits
```

That works in an isolated git worktree under `~/.nightowl/worktrees/`. Your
branch is never touched, and if the worktree can't be created the run *aborts*
rather than falling back to something riskier. It never commits, pushes, or
tags.

Review afterwards with the command nightowl prints for you:

```bash
git -C ~/dev/invoice-parser diff nightowl-aaaa1111
```

---

## What happens during a run

You don't write instructions. nightowl reads the todo list Claude was already
keeping in that session, builds the prompt from it, and reopens the same
conversation so the context is still there.

While it runs, it gets budget reminders at 50%, 75% and 90% spent — each one
telling it to narrow scope rather than start something new. At 90% it stops and
writes a short handoff note: what it did, what it didn't, the next step.

If it hits the ceiling it gets killed. That's the ceiling working, not a
failure.

---

## Getting told when it's done

nightowl runs while you're away — so a run that finishes silently at 2am is
indistinguishable from one that never started, until you happen to check.

```bash
nightowl notify           # what's configured
nightowl notify --test    # send one now
```

**Desktop notifications work with no setup** (macOS `osascript`, Linux
`notify-send`).

**To get them on your phone**, use [ntfy](https://ntfy.sh) — no account
needed. Install the app, pick a topic name, and add it to
`~/.nightowl/config.json`:

```jsonc
{ "notify": { "ntfy": "https://ntfy.sh/your-unique-topic-name" } }
```

Pick something unguessable — anyone who knows the topic can read it.

Or POST the run summary anywhere with `"webhook": "https://..."` — Slack,
Discord, Home Assistant.

You get told when a run **finishes**, when it **uses up its budget** and stops
at the ceiling, and when it **fails to launch**. Nothing is sent for dry runs;
a notification for something that didn't happen just teaches you to ignore
notifications.

Delivery can never affect a run. Every send is timeboxed and its failure
swallowed — a flaky network will not turn a successful run into a failed one.

### Watching from your phone

Anthropic's mobile apps are worth having alongside this. Claude Code sessions
started on your machine can be monitored and steered from the **Claude iOS and
Android apps**, so when nightowl notifies you that a run finished, you can read
the handoff note and keep going from wherever you are rather than waiting until
you're back at your desk.

The pairing that works: nightowl decides *what* to work on and tells you when
it's done; the mobile app is how you check the result and reply.

---

## Your first week

Three commands worth running once you have some history.

**Did it pick well?**

```bash
nightowl eval
```

Replays your own history and checks the ranking against what you actually did —
including against dumb baselines like "just pick the most recent". If nightowl
can't beat those, it says so plainly. This is the most useful output the project
produces, especially when it's bad news.

**Did you keep the work?**

```bash
nightowl outcomes
```

Infers `adopted` / `discarded` / `ignored` from what happened to each worktree.
No feedback form.

**Were the cost estimates right?**

```bash
nightowl calibrate           # show the analysis
nightowl calibrate --apply   # use it
```

Compares predicted against actual cost across your runs and corrects the
estimator.

---

## Telling it what to ignore

nightowl occasionally asks — at most once a fortnight, and only about a stalled
session where the repo gives no clue either way. You can also just tell it:

```bash
nightowl verdict aaaa1111 dead      # never suggest this again
nightowl verdict aaaa1111 live      # I'm still on this
nightowl verdict aaaa1111 snooze    # not this week
nightowl verdicts                   # what you've ruled on
```

`dead` is permanent. This is how you correct a bad pick — no config editing.

---

## Running it unattended

```bash
nightowl watch --go
```

Polls, and does nothing until your window is nearly over with quota unspent.

It works, but use the manual version for a few weeks first, until you've seen
enough of its picks to trust them. Reading a dry run costs nothing.

---

## Install as a Claude Code plugin

Adds `/nightowl` in chat, a quiet nudge when quota is about to expire, and the
budget reminders during runs.

```
/plugin marketplace add llamaoat/nightowl
/plugin install nightowl@nightowl-marketplace
```

---

## All commands

| Command | What it does |
|---|---|
| `nightowl` | Dashboard (same as `status`) |
| `nightowl explain <id>` | Full scoring breakdown for one session |
| `nightowl run [id]` | Dry run; add `--go` to actually spend |
| `nightowl watch` | Fire only when quota is about to expire |
| `nightowl verdict <id> <v>` | Record `live` / `dead` / `snooze` |
| `nightowl outcomes` | What you did with past runs |
| `nightowl calibrate` | Fix cost estimates from predicted vs actual |
| `nightowl eval` | Measure the ranking against your real history |
| `nightowl notify` | Notification setup; `--test` sends one now |

**Useful flags**

| Flag | Meaning |
|---|---|
| `--window 5d` | How far back to look: `5d`, `1w`, `2w`, `30d`, `all` |
| `--show-excluded` | List what was filtered out, and why |
| `--include-research` | Also consider read-only research sessions |
| `--go` | Actually spend. Without it, always a dry run |
| `--allow-edits` | Permit file edits, in an isolated worktree |
| `--terse` | Ask for concise output during the run |
| `--json` | Machine-readable output |

---

## Safety, briefly

All of these are on by default and have to be turned off deliberately:

- Nothing is spent without `--go`
- No files are edited without `--allow-edits`
- Edits happen in a detached git worktree, never your branch
- 25% of the 5-hour window and 20% of the week are never spendable
- A token ceiling, enforced by killing the run (see the caveat below)
- 45-minute wall-clock timeout
- No commits, pushes, or tags, ever
- Below score 25 it refuses to pick at all

`--dangerously-skip-permissions` is unreachable from every code path.

**One honest caveat about the ceiling.** It is checked after each metered turn,
so a run can overshoot by up to one turn's cost — there is no way to cancel a
response before it arrives. In practice that is a few thousand tokens. On a very
small budget the overshoot can be proportionally large, so do not treat the
ceiling as a guarantee of an exact spend.

---

## Configuration

Optional. Lives at `~/.nightowl/config.json` — see `config.example.json`.

```jsonc
{
  "window": "2w",
  "reserve": { "blockPct": 0.25, "weeklyPct": 0.20 },
  "run": { "allowEdits": false, "useWorktree": true, "timeoutMinutes": 45 },
  "ignore": ["/work/client-repos"]
}
```

Silence the occasional question with `NIGHTOWL_ASK=off`, and the session-start
nudge with `NIGHTOWL_NUDGE=off`.

---

## Other agent CLIs

This release supports **Claude Code only**.

The internals are already CLI-agnostic — the ranking model, repo archaeology,
chat-vs-work classification, eval harness and calibration all operate on a
normalised session record. Adding another CLI means writing one descriptor in
`lib/adapters/index.mjs`, not touching the ranking.

A Gemini CLI adapter is on the `adapters` branch. It is deliberately not in the
main release: it was written from published docs rather than tested against
real transcripts, and an adapter that silently produces wrong numbers is worse
than no adapter.

## Things to know before relying on it

**Quota numbers are estimates.** There is no public API that reports a
subscription's remaining tokens. Everything is inferred from your own
transcripts and calibrated against your own history. It reliably answers *"do I
have room, and is it about to reset?"* It is not a billing oracle.

**Claude Code only.** Your claude.ai chat history isn't reachable from your
machine — no local store, no read API. If your unfinished work lives in web
chats, nothing here can see it.

**Claude Code already auto-continues limit-interrupted sessions**, on by
default. nightowl deliberately stays out of the way for recent limit-stops and
focuses on everything else: old stalls, and sessions that stopped because you
closed the laptop.

**The ranking is tuned on synthetic fixtures, not real data.** It will be wrong
for someone's workflow. `nightowl eval` exists so you can find out whether it's
wrong for yours.

**POSIX only.** Linux and macOS, tested on Node 18/20/22.

---

## Contributing

The most valuable thing you can send is `nightowl eval` output from real
history — **especially if nightowl ties or loses to a baseline.** After that, a
failing fixture for a bad ranking beats any heuristic tweak.

See [CONTRIBUTING.md](CONTRIBUTING.md). [DESIGN.md](DESIGN.md) explains why
things work the way they do.

## License

MIT
