# README rewrite notes

Notes for updating `README.md` by hand. Nothing here has been applied to the README — it's untouched.

## Repo and layout

- Repo renamed `agent-continuity` → `continuity`. Any clone URL or title referencing the old name is stale.
- Skills moved: `scaffold/` → `claude/skills/scaffold/`, same for `orient/` and `preserve/`. `claude/` is the plugin root; repo-root `.claude-plugin/marketplace.json` is the marketplace catalog.
- New: `example/` (a fictional TypeScript webhook service showing filled-in `state.md`, `decisions.md`, and a real audit run), `LICENSE` (MIT, 2026 Anthony DeLuise).

## Install section — replace entirely

- The current symlink instructions are obsolete on two counts: the paths changed, and this is a plugin now.
- New install:
  ```
  /plugin marketplace add adeluise/continuity
  /plugin install continuity@continuity
  ```
- Dev / local alternative, for hacking on the skills without installing:
  ```bash
  claude --plugin-dir ./claude
  ```
- Symlinking still works if someone wants it, but the source paths are now `claude/skills/<name>` — probably not worth documenting alongside the plugin path.

## Skill names are namespaced

- Plugin-installed skills invoke as `/continuity:scaffold`, `/continuity:orient`, `/continuity:preserve`, `/continuity:audit`. The bare `/scaffold` form only works for a personal-directory install.
- The README currently uses bare `/scaffold` etc. throughout — decide whether to show the namespaced form everywhere or note the namespacing once up front. (The skills themselves no longer refer to each other by slash-command name, for this reason.)

## Fourth skill: audit

- Needs its own section, same shape as the other three.
- Checks each active decision in `decisions.md` against the codebase and git history. Reports three buckets — holding, violated, superseded candidate — plus "cannot verify" when the evidence isn't in the repo.
- Read-only except for one thing: it can flip a decision's `Status` to `superseded`, and only after you confirm.
- `example/audit-output.md` is a full sample run and is probably the fastest way to show what it does.

## Behavior changes to mention

- **Status field on decisions.** Entries now carry `**Status:** active` or `superseded`. Missing field = active. The Status line is the only part of an existing entry that may ever be edited — the log is otherwise still append-only.
- **preserve harvests `scratch.md`.** Still-relevant open questions and gotchas migrate into the matching `state.md` sections before the wipe, and anything discarded is listed explicitly. The README's "scratch.md — wiped clean" line undersells it now.
- **preserve offers to commit.** After presenting the result it asks once whether to commit `state.md` and `decisions.md`. It's the only question preserve asks.
- **orient warns on an uncommitted `state.md`.** The staleness check anchors on the last commit touching `state.md`, so a dirty working copy makes it unreliable — orient now says so instead of reporting a confident wrong answer.

## Also

- Point at `example/` somewhere near the top — it explains the system faster than the prose does.
- Mention the MIT license.
