# continuity

Coding agents forget between sessions. Losing where you left off is annoying but losing *why* you chose something, and what you rejected to get there, is expensive.

Continuity keeps both in the repo: three markdown files, four skills that maintain them.

| File | Purpose |
|------|---------|
| `decisions.md` | Append-only log of major decisions: what, when, why, what was rejected (tracked) |
| `state.md` | Context bridge between sessions, fully replaced at the end of each one (tracked) |
| `scratch.md` | Ephemeral working notes, harvested and wiped at the end of each session (gitignored) |

`decisions.md` is the durable half. Written decision records are an old idea that mostly failed in practice — not because writing them was hard, but because nothing ever read them. Here something does: the log goes into context every session, and `audit` checks each entry against the code that actually shipped.

It's all plain markdown in the repo, so it survives a fresh clone, a new machine, or a teammate — and anything can read it, not just Claude Code.

The fastest way to understand the system is [`example/`](example/): a fictional webhook service with a filled-in `state.md`, eight months of `decisions.md`, and a verbatim audit run showing what drift looks like when it's caught.

## What this is not

- **Not `claude --resume`.** That replays a raw transcript — machine-local, token-heavy, noise included. Continuity distills a session into curated, human-readable files versioned alongside the code.
- **Not a PRD or design doc.** Those capture intent up front and drift from the day they're merged. `state.md` is operational now-state, rewritten every session; `decisions.md` is rationale accumulated over time.
- **Not CLAUDE.md.** CLAUDE.md is standing instructions that rarely change. Session state churns; folding it in bloats every prompt with things that were true last Tuesday. The split is what's always true vs. what's true right now.
- **Not memory machinery.** No MCP server, no vector store, no embeddings, no schema. These are rejections, not omissions. The governing principle: state is legible and lives in the repo — plain markdown, versioned in git, correctable in a text editor, zero runtime dependencies.

## Skills

| Skill | When | What it does |
|-------|------|--------------|
| `scaffold` | Once per project | Creates the three files, updates `.gitignore` and `CLAUDE.md`, installs the session-start hook |
| `orient` | Start of a session | Reads the files and recent git, presents where you are, flags staleness |
| `preserve` | End of a session, before `/clear` | Rewrites `state.md`, appends decisions, harvests and wipes `scratch.md`, offers one commit |
| `audit` | When the decision log feels stale | Checks each active decision against the code and git history, reports drift |

## Install

In Claude Code:

```
/plugin marketplace add adeluise/continuity
/plugin install continuity@continuity
```

Plugin-installed skills are namespaced: they invoke as `/continuity:scaffold`, `/continuity:orient`, and so on. This README uses the short names.

To hack on the skills without installing, run Claude Code from a clone with the plugin loaded directly:

```bash
claude --plugin-dir ./claude
```

## scaffold

Run once per project. Creates whichever of the three files don't exist (never overwrites), adds `scratch.md` to `.gitignore`, and appends a short context-system block to `CLAUDE.md` — creating a minimal one if the project has none. It also installs a SessionStart hook in `.claude/settings.json` that injects `state.md` into context on startup and after `/clear`, so every session begins oriented without invoking anything. Deterministic — it asks no questions.

Invoke `/continuity:scaffold`, or naturally: "Set up the context system for this project."

## orient

The deliberate version of the session-start hook: reads `state.md`, `decisions.md`, `scratch.md`, and recent git activity, then presents a structured summary with the next step front and center. It checks staleness by listing commits made since `state.md` was last committed — and if `state.md` itself has uncommitted changes, it says the check is unreliable rather than reporting a confident wrong answer. Read-only; it ends by waiting for direction, never by starting work.

Invoke `/continuity:orient`, or naturally: "Let's pick this back up."

## preserve

Run at the end of a session, before `/clear`. Rewrites `state.md` in full — where you ended, what's working, what's broken, landmines, and the first thing to do next session. Appends decisions to `decisions.md` only when they clear a threshold (a door closed, something explicitly rejected, rationale that won't be obvious later); each new entry carries `**Status:** active`. The log is append-only with one exception: if a decision made this session directly reverses an earlier entry, that entry's Status flips to `superseded` — that line, nothing else. An entry with no Status line is active.

Before wiping `scratch.md`, preserve harvests it: still-relevant open questions, gotchas, and unfinished threads migrate into the matching `state.md` sections, and every note it discards is listed explicitly so nothing disappears silently. It presents the result for review, then asks exactly one question: commit `state.md` and `decisions.md`? Yes or no — committing stays your call.

Invoke `/continuity:preserve`, or naturally: "Let's wrap up."

## audit

Checks every active decision in `decisions.md` against the codebase and git history — superseded entries are skipped as history. Each finding lands in one of four buckets: **holding**, **violated** (the code contradicts a decision that's still supposed to stand — the drift is the bug), **superseded candidate** (the project deliberately moved past the decision, but nothing recorded it), or **cannot verify** (the evidence isn't in this decision that's still supposed to stand — the drift is the bug), **superseded candidate** (the project deliberately moved past the decision, but nothing recorded it), or **cannot verify** (the evidence isn't in this repo — absence of evidence is never treated as violation). Every claim cites a file path, symbol, or commit  migrate into the matching `state.md` sections, and every note it discards is listed explicitly so nothing disappears silently. It presents the result for review, then asks exactly one question: commit `state.md` and `decisions.md`? Yes or no — committing stays your call.

Invoke `/continuity:preserve`, or naturally: "Let's wrap up."

## audit

Checks every active decision in `decisions.md` against the codebase and git history — superseded entries are skipped as history. Each finding lands in one of four buckets: **holding**, **violated** (the code contradicts a decision that's still supposed to stand — the drift is the bug), **superseded candidate** (the project deliberately moved past the decision, but nothing recorded it), or **cannot verify** (the evidence isn't in this repo — absence of evidence is never treated as violation). Every claim cites a file path, symbol, or commit hash.

It's read-only with one exception: for superseded candidates it proposes the flip and, only after you confirm, edits that entry's Status line to `superseded` — nothing else in the file. See [`example/audit-output.md`](example/audit-output.md) for a full run.

Invoke `/continuity:audit`, or naturally: "Do our past decisions still hold?"

## Caveats

`state.md` is tracked so its history is versioned, but the system is currently optimized for a single user: preserve does a full replacement, so expect merge conflicts if multiple contributors preserve on the same branch. Will find a solution when there's a need.

MIT licensed.
