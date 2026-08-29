# nightowl — design notes

Why the thing works the way it does. If you just want to use it, the
[README](README.md) is shorter and friendlier; this is the reasoning behind the
decisions, written down so contributors can argue with it.

---

## Why this exists (and what already exists)

There is a crowded field of Claude usage tools. Before building, check whether
one of these already solves your problem — several are excellent and far more
mature:

| Tool | What it does | Overlap |
|---|---|---|
| [`ccusage`](https://github.com/ryoppippi/ccusage) | Historical token/cost reports from local JSONL | Reads the same data source |
| [`Claude-Code-Usage-Monitor`](https://github.com/Maciek-roboblog/Claude-Code-Usage-Monitor) | Live burn-rate dashboard, P90 limit auto-detection | The calibration idea is borrowed from here |
| `claude-auto-retry` and similar | Wait for a limit to reset, then resume | Resumption, but reactive |
| Claude Code's own auto-continue | Resumes your session when your limit resets | Ships in the product now |
| [`claude-todos`](https://github.com/eranhirs/claude-todos) | Discovers todos across sessions, spawns runs | Closest neighbour |

### Claude Code now ships auto-continue

As of v2.1.234, Claude Code resumes an interrupted session automatically once
your usage limit resets — **on by default**. That covers the single case
nightowl used to score highest, so it is worth being precise about what is left.

The triggers are opposites. Auto-continue fires on **scarcity**: you hit the
wall. nightowl fires on **surplus**: you didn't. If you never hit a limit,
auto-continue never runs.

|  | auto-continue | nightowl |
|---|---|---|
| Fires when | you hit the wall | you didn't hit the wall |
| Looks at | the session that was live | every stalled session |
| Reaches back | the interrupted turn | weeks |
| Covers stalls from | usage limits | limits, closed laptops, meetings, boredom |
| Decides | nothing — one option | which one, and whether any qualifies |

Most stalled work was never limit-interrupted. You closed the laptop, got
pulled into a meeting, lost interest. Nothing in the product touches those.

**The scorer accounts for this explicitly.** A limit-stop from the last two
days gets *no* boost — auto-continue is probably already handling it and we
should stay out of the way. One from two weeks ago keeps the boost, because
auto-continue had its chance and the work is still unfinished, which makes it
more interesting rather than less. Non-limit interrupts (Esc, closed laptop)
keep the boost outright; they were never covered.

**What none of them do:** treat *unused* quota as the trigger. Every existing
tool is organised around scarcity — warning you before you hit the wall, or
waiting out a wall you already hit. `nightowl` is organised around surplus, and
around the harder question that follows:

> Given exactly this much budget, which unfinished thing can it *actually
> finish*, without a human in the loop, without me regretting it?

Answering that is the whole project. The dashboard is the easy part.

---

---

## How the ranking works

```
score = finishability × autonomy × fit × 100
```

Multiplied, not summed — a zero on any axis disqualifies. A confident wrong
pick costs real quota, so the model is built to abstain rather than guess.

**finishability** — how close to done, from the session's last `TodoWrite`
state. Peaks around 70–90% complete: far enough along that finishing is
plausible, not so far that nothing is left. Multiplied up ×1.6 if the session
stopped at a usage limit (a strong "interrupted, not abandoned" signal),
down ×0.25 if the last message reads like a sign-off. Decays with a 3-day
half-life, because stale context is usually wrong context.

**autonomy** — can it proceed without you? A session parked on *"Redis or
Postgres?"* scores ×0.15 here no matter how close to done it is. Remaining
todos mentioning credentials, deploys, or migrations get ×0.35. Mechanical work
— tests, docs, types, lint — gets ×1.25. **This is the axis most obvious
implementations of this idea forget**, and it is the one that decides whether
unattended runs are useful or infuriating.

**fit** — does the estimated remaining cost fit the spendable budget? Cost is
estimated from the session's *own* observed tokens-per-completed-todo, which
beats any global constant: a session doing heavy file reads genuinely costs
more per task than one editing markdown.

Run `nightowl explain <id>` to see the arithmetic for any candidate.

---

---

## The budget tracker

`nightowl run` meters the child process line by line from `stream-json`. Until
v0.2 it used that number for exactly one thing: killing the run on breach. The
model itself was told the budget once, in the opening prompt, and then never
heard about it again — it spent the whole run flying blind.

Two results say that's the wrong shape:

- **BudgetThinker** ([arXiv 2508.17196](https://arxiv.org/abs/2508.17196)) —
  stating a budget once in the prompt is insufficient; the model needs to be
  continuously reminded of what remains as it works.
- **Budget-Aware Tool-Use / BATS** ([arXiv 2511.17006](https://arxiv.org/abs/2511.17006), Google) —
  simply granting a larger budget doesn't improve agent performance, because
  agents lack budget awareness and hit a ceiling. A lightweight *Budget Tracker*
  that keeps remaining budget in context fixes this, and lets the agent decide
  whether to dig deeper on a promising lead or pivot.

### How it works

The runner can't talk to the model mid-turn — it only reads stdout. A hook can,
but it has no idea what the run's budget is. Neither half works alone, so they
join through a small state file:

```
runner.mjs  ──writes──▶  ~/.nightowl/active-run.json  ──reads──▶  hooks/budget.mjs
(meters stream-json)      { budget, spent, phase }       (PostToolUse hook)
                                                                  │
                                                    hookSpecificOutput
                                                     .additionalContext
                                                                  ▼
                                                        the model's context
```

Claude Code wraps that string in a system reminder and inserts it at the point
the hook fired. The model reads it on the next request; it never appears as a
chat message. At launch, `--append-system-prompt` tells the model the reminders
are coming and to treat them as a planning input rather than something to
reply to.

### The four phases

Each reminder carries a decision rule, not just a number. "You have 40% left"
invites no particular behaviour; "finish what's open, start nothing new" does.
That's the dig-deeper-vs-pivot instruction from BATS, in four steps:

| Spent | Phase | What the model is told |
|---|---|---|
| 0–50% | `explore` | Investigating broadly is fine. Dig into promising leads. |
| 50–75% | `consolidate` | Finish what's open. Start nothing new. Pivot now if the lead isn't paying off. |
| 75–90% | `wrap-up` | Close the smallest genuinely finishable unit. Stop reading new files. Be terse. |
| 90%+ | `land` | Stop starting anything. Write the handoff. Stop. |

### Why it doesn't eat the budget it protects

A tracker that injected on every tool call would be self-defeating: at ~45
tokens across 200 tool calls it would spend 9k describing itself. So injection
fires only on **phase transitions** — three times per run, maximum — with a
45-second minimum interval as a backstop. Measured overhead is **~163 tokens
per run**, about 0.5% of a 30k budget. Silence is the default and the common
case.

The `PostToolUse` hook is installed globally, so it fires in every session on
the machine. Its guard chain, cheapest check first: no state file → silent;
different `session_id` → silent; runner died (stale timestamp) → silent and
clean up; no phase crossed → silent. It never exits non-zero, because exit
code 2 would block the tool loop, and a budget reminder isn't worth blocking
anyone's work over.

### Flags

```bash
nightowl run --go                # tracker on by default
nightowl run --go --terse        # also enforce concise output for the whole run
nightowl run --go --no-tracker   # opt out entirely
```

`--terse` is the smaller half of this. It only touches output tokens, and in
agentic runs tool results and context dominate spend — expect 20–30% off a
minority slice. The phase guidance is what actually changes outcomes, because
it changes what the model *attempts*, not just how it writes.

### Honest caveat

This is ported from published results on other agent scaffolds; it has not been
A/B'd on Claude Code with a real task corpus. The phase thresholds (50/75/90)
and the exact wording are informed guesses. `~/.nightowl/runs.jsonl` records
`budgetTokens`, `spent`, `killedForBudget`, and injection count for every run —
enough to actually measure whether this helps. If you gather that data, a PR
with numbers would be worth more than any amount of further tuning.

---

---

## Chats vs work

Most of your session history isn't work. It's "how does this library handle
retries", "explain this stack trace" — conversations with nothing to finish.
Offering to resume those wastes your attention on candidates that were never
real.

The test is simple and free: **a chat modifies nothing; work modifies files.**
Claude Code records every `Edit`, `Write`, and `NotebookEdit` call in the
transcript, so the distinction is already there.

| Tier | Test | Eligible |
|---|---|---|
| `work` | edited a file, or has a 2+ item todo list | yes |
| `research` | substantial reading, nothing written | `--include-research` |
| `chat` | nothing touched, no plan | never |

Bash counts too — a session that did all its editing through `mv`, `tee`, or
`git apply` is work, not chat.

**A plan is an artefact.** A session with no edits but a real multi-item todo
list still counts as work. That's what rescues "we designed the thing but never
started building it".

**What this gets wrong:** a long architecture discussion with no edits *and* no
todo list looks like `research` and gets binned. Not fixable without asking. So
nothing is dropped silently:

```bash
nightowl status --show-excluded
```

```
  excluded (3 sessions)

  research · read-only sessions — resuming means re-reading
      · Claude usage limit reached. Resets at 9pm — 4 read/search calls, nothing written

  chat · conversations — nothing was ever produced to finish
      · and microtasks? — 4 turns, nothing touched, no plan
```

That's the list to *read*, not one to fill in. Misclassification is cheap to
correct after the fact — `nightowl verdict <id> dead` and it never returns.

---

---

## Repo evidence: what happened *after* the transcript

A transcript is a snapshot of the past. Read alone, a session you abandoned
because the idea was bad looks identical to one you abandoned because your
quota ran out — same half-finished todo list, same abrupt ending.

But the repo knows. Before scoring, `nightowl` asks git what happened since:

| Evidence | Verdict | Effect |
|---|---|---|
| Remaining todos appear in later commit subjects | `superseded` | ×0.05 — the work landed without Claude |
| No commits in 60+ days | `dead-repo` | ×0.1 |
| The session's branch no longer exists | `branch-gone` | ×0.15 |
| Most edited files are gone | `files-gone` | ×0.2 |
| Commits since, touching this session's files | `moved-on` | ×0.75 — context has drifted |
| Uncommitted work still in the tree | `in-flight` | ×1.15 |
| Nothing at all since | `silent` | ×1 — **genuinely ambiguous** |

Todo-to-commit matching is word-overlap on significant terms (60% threshold,
filler words dropped). It's deliberately conservative: a false `superseded`
silently hides work you still want, which is worse than leaving a dead session
in the list.

All deterministic, all free, no model call and no question to you. Disable with
`--no-repo`.

**What it can't do:** non-git projects get nothing, uncommitted work is
invisible, and squashing or rewording defeats subject matching. It narrows the
ambiguous band rather than eliminating it. The residue is what the question is
for.

---

---

## The one question

Only the `silent` verdict is ambiguous — a stale session in a live repo where
nothing has happened either way. Git can't settle it and neither can a
heuristic, so that's the one case worth a human keystroke.

It rides on the `Stop` hook, which fires when Claude finishes a turn with the
conversation still open. (`SessionEnd` can't block termination, so anything it
asked would arrive after you'd gone.) Claude asks in chat, once, plainly.

Every condition must hold before you're bothered:

1. a candidate is older than `staleDays` (default 7)
2. its repo evidence is `silent`
3. that session has never been asked about
4. nothing has been asked in the last 24 hours
5. the answer would change something — it scores well enough to be a plausible
   pick if you vouch for it

In the test suite, **one of five evidence cases** reaches the user. In practice
expect a question every week or two. If it's more often than that, the gate is
wrong and it's a bug worth filing.

Answers are permanent:

```bash
nightowl verdict <id> dead     # never suggested again, at any score
nightowl verdict <id> live     # staleness decay suspended for 14 days
nightowl verdict <id> snooze    # hidden for a week
nightowl verdicts               # everything you've ruled on
```

Silence `NIGHTOWL_ASK=off`. Tune with `NIGHTOWL_STALE_DAYS`,
`NIGHTOWL_ASK_COOLDOWN_H`.

The design rule: **ask at the point of value, never upfront.** A `triage`
command that walks you through 40 stale sessions would be a chore presented
exactly when you're least motivated — these are things you already walked away
from once. One question, about one session, when it's about to matter.

---

---

## Measuring whether any of this works

The ranking heuristics are guesses. `nightowl eval` replays your own history and
checks them against what you actually did.

```bash
nightowl eval                    # replay, compare against baselines
nightowl eval --horizon 7        # tighter definition of "came back to it"
nightowl eval --json             # full per-moment detail
```

**The labels already exist.** For every session that stalled, your history says
whether you came back. That is exactly what the score predicts. No survey, no
experiment, no waiting.

At each stall point T, the harness reconstructs what `nightowl` would have seen
at T, ranks the pool, then looks forward: did you resume the top pick within the
horizon?

```
  ranker               P@1          95% CI    P@3    MRR
  nightowl            100%        85%–100%   100%   1.00
  most recent          38%         21%–59%    76%   0.60
  most todos left       0%          0%–15%     0%   0.08
  fewest todos left   100%        85%–100%   100%   1.00
```

**Baselines are the point.** "The model got 100%" means nothing on its own. The
question is whether it beats *sorting by one dumb signal*. In the run above it
doesn't — it ties `fewest todos left` — and the tool says so rather than
reporting the 100% and stopping. If your ranking can't beat one line of
sorting, the ranking is decoration and should be deleted.

### Three things that make the numbers honest

**No hindsight.** `repoEvidence` normally runs `git log --since=sessionEnd` with
no upper bound. During a replay that sees commits made *after* T, so the model
would "discover" a todo was superseded by a commit that hadn't happened yet and
report excellent, meaningless results. Every query is bounded by `--until=T`,
and there's a control test asserting the future is visible without the cutoff
and invisible with it.

**Present-tense signals are disabled, not faked.** Uncommitted files and branch
existence can't be reconstructed for a past moment, so during a replay they're
switched off. Eval results are therefore a slight *under*-estimate of live
performance — the safe direction to be wrong in.

**Right-censoring.** A session that stalled two days ago has no label yet; you
might resume it tomorrow. Scoring it as a negative would systematically punish
recency, the signal the model weights most. Those moments are excluded and
counted separately.

### Confidence intervals, not percentages

Every P@1 carries a Wilson interval. On 20 moments, 60% means 39–78% — a range
wide enough that "the model leads" usually isn't yet evidence. The tool says
"the intervals overlap" rather than declaring a winner.

### What this can't measure

"Did you resume it" is not "was it worth resuming." You may have failed to come
back to something because you *forgot* — the precise failure this tool exists to
fix. Optimising hard against this label trains it toward redundancy:
recommending only what you'd have done anyway. The upside case is not
retroactively measurable.

That's tolerable because today's failure mode is suggesting garbage, not missing
gems, and the negatives are clean. Read the numbers as an upper bound on
badness, not a measure of value.

### Don't fit fifteen constants on fifty samples

`score.mjs` has ~15 hand-tuned numbers. A typical history has a few dozen
labelled moments. Fitting them all would overfit instantly and produce something
worse than the guesses. Do sensitivity analysis first — perturb each constant,
see which actually move the ranking. Probably three or four dominate (recency
half-life, interrupted multiplier, autonomy penalty). Fit those, hold out a
third of the history, leave the rest alone.

---

---

## Learning from what you actually did

Two feedback loops, neither of which asks you anything.

### `nightowl outcomes` — did you keep the work?

Every run with edits lands in an isolated worktree. What happens to that
worktree afterwards *is* the review:

| Signal | Outcome |
|---|---|
| The repo gained commits touching the files the run changed | `adopted` |
| Worktree deleted, nothing matching landed | `discarded` |
| Worktree still sitting there untouched after 5 days | `ignored` |
| Less than a day old | `pending` |

A feedback button collects data from the few who bother to press it — and the
people most annoyed are the least likely to stop and fill in a form. Reading
git gets a label on every run instead.

**These labels are weak and directional.** Because the runner forbids
committing, the branch holds uncommitted changes, so `git branch --merged` is
useless and adoption has to be inferred from the original repo gaining commits
that touch the same files. `adopted` means "this run's output plausibly
survived", not "your patch was merged verbatim". The tool says so in its own
output rather than quietly implying more.

### `nightowl calibrate` — was the cost estimate right?

This is the one place where "the tool improves itself" is honest, because the
ground truth is unambiguous and free. Every run already records `estTokens`
(predicted) and `spent` (actual).

```bash
nightowl calibrate              # show the analysis, change nothing
nightowl calibrate --apply      # use it
nightowl calibrate --reset      # back to defaults
```

If it habitually predicts 30k for jobs that take 80k, that isn't a matter of
taste — it's a measurable bias with an obvious correction.

**Censoring is the detail that makes this correct.** When a run is killed at
the budget ceiling, `spent` is not the true cost — it's a lower bound. Averaging
those rows in would drag the factor *down*: the tool would conclude it
overestimates, shrink its estimates, and hit the ceiling even more often. A
feedback loop that makes the problem worse. So censored runs are excluded from
the ratio, and if more than half your runs hit the ceiling the tool refuses to
calibrate at all and tells you to raise the budget instead.

Other guard rails:

- **Median, not mean.** One agent stuck in a retry loop can be 20× the others.
- **Clamped to 0.4×–3.0×.** A factor outside that is more likely a bug than a
  real bias, and applying it would make every future estimate worse. Clamped on
  read too, so hand-editing the file can't break estimation.
- **Minimum 5 completed runs.** A factor fitted on three runs is noise.
- **Per-method factors.** The three estimation paths have different biases;
  lumping them averages the signal away.
- **Never applied automatically.** `--apply` is a separate, explicit step.

### What is deliberately *not* built

No automatic rewriting of prompts or scoring weights. With twenty runs, fitting
anything isn't learning, it's memorising noise — into a tool that runs
unattended with write access to your repos. And it drifts silently: nobody
notices until the picks get strange, and by then there's no clean version to
diff against.

If self-tuning of the ranking weights ever lands, it should end in a
*proposal* — validated against `nightowl eval` on held-out history — that you
read and commit yourself.

---
