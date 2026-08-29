# Security

## Threat model

`nightowl` can spawn an agent with write access to your repositories, using your
credentials, with no human watching. That is the whole feature, and it is also
the whole risk.

Controls, all on by default and all opt-out:

- `--go` is required before anything is spent
- `--allow-edits` is required before any file is written
- edits run in a detached git worktree under `~/.nightowl/worktrees/`, never on
  your branch; if the worktree cannot be created the run aborts
- no code path can reach `bypassPermissions` or `--dangerously-skip-permissions`
- the runner never invokes `git commit`, `git push`, or `git tag`

## Trusted files

`~/.nightowl/active/*.json` is read by a hook that injects text into a model's
context. The injected string is assembled from numeric fields plus constants in
`lib/tracker.mjs`, so a tampered state file cannot inject arbitrary prompt text
— but treat that directory as trusted regardless, and do not make it
world-writable.

## Reporting

Open a draft security advisory on GitHub rather than a public issue.
