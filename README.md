# continuity

Claude Code forgets between sessions: major decisions made, the reasoning you'd made, what you ruled out, where you left off thinking. `/clear` wipes all of it, and the next session starts cold.

Three Claude Code skills close that gap. They maintain three files in your project — `state.md`, `decisions.md`, `scratch.md` — that capture where you ended, what was decided, and what's next.

## Use it

- `/scaffold` — once per project. Creates the three files.
- `/orient` — at the start of a session. Reads the files and recent git, summarizes where you are.
- `/preserve` — at the end of a session, before `/clear`. Captures the session so the next one can pick up cold.

## Install

Clone the repo, then symlink each skill into your Claude Code skills directory from the repo root:
```bash
ln -s "$(pwd)/scaffold" ~/.claude/skills/scaffold
ln -s "$(pwd)/orient" ~/.claude/skills/orient
ln -s "$(pwd)/preserve" ~/.claude/skills/preserve
```

---

# scaffold

Scaffolds the three-file persistent context layer for working with Claude Code across sessions.

| File | Purpose |
|------|---------|
| `decisions.md` | Append-only log of major decisions: what, when, why, and what was rejected |
| `state.md` | Context bridge between sessions — replace before ending so the next session can pick up cold |
| `scratch.md` | Ephemeral working notes, ideas, open questions — wiped between sessions |

These sit alongside `CLAUDE.md` in the project root. `/scaffold` creates a minimal `CLAUDE.md` if one doesn't exist. It also installs a SessionStart hook into `.claude/settings.json` that injects `state.md` into context on session start and after `/clear`, so every session begins oriented without invoking anything. `/orient` remains the deliberate version — synthesis plus a staleness check against git.

`state.md` is tracked so its history is versioned. Currently optimized for single user, expect merge conflicts with multiple contributors since `/preserve` does a full replacement. Will find a solution when there's a need.

## Usage

Invoke directly:

```
/scaffold
```

Or trigger it naturally:

```
Set up the context system for this project
```

It checks what exists and sets up what's missing.

---

# orient

The complement to preserve. Reads `state.md`, `decisions.md`, and recent git activity, then presents a concise orientation summary. Shows you where things stand and waits for direction.

## Usage

Invoke directly:

```
/orient
```

Or trigger it naturally:

```
Let's pick this back up
```

It reads the context files and git history, synthesizes a summary with the next step front and center, and waits for you to say what to work on.

---

# preserve

The complement to orient. Runs at the end of a session before `/clear` to update the three context files so the next session can start cold.

- **state.md** — overwritten with where you ended, what's working, what's broken, landmines, and the first thing to do next session
- **decisions.md** — appends significant decisions (if any were made) with rationale and rejected alternatives
- **scratch.md** — wiped clean

## Usage

Invoke directly:

```
/preserve
```

Or trigger it naturally:

```
Let's wrap up
```

It reads the session context and git diff, writes the files, and shows you the result for a quick review before you `/clear`.

---
