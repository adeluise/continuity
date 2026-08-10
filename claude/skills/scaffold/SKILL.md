---
name: scaffold
description: "Scaffolds the three-file context system (decisions.md, state.md, scratch.md) into a project. Use when setting up a new project's context layer, starting a context system, or when the user wants session context files alongside CLAUDE.md."
allowed-tools: Read, Write, Edit
---

# Scaffold

Scaffold the three-file context system into the current project. These files sit alongside `CLAUDE.md` as the persistent context layer for working with Claude Code across sessions.

## Existing files

!`ls decisions.md state.md scratch.md .gitignore CLAUDE.md .claude/settings.json 2>/dev/null || echo "(none found)"`

## Files

| File | Purpose | Git |
|------|---------|-----|
| `decisions.md` | Append-only log of major decisions | Tracked |
| `state.md` | Context bridge between sessions | Tracked |
| `scratch.md` | Ephemeral working notes for the current session | Ignored |

## Rules

0. **This is a deterministic scaffold — no questions needed.** When invoked, immediately check what exists and create what's missing. Do not ask for project type, stack, or preferences.
1. **Never overwrite existing files.** If `decisions.md`, `state.md`, or `scratch.md` already exists, skip it and tell the user.
2. **Always update `.gitignore`.** Append `scratch.md` if it's not already listed. Create `.gitignore` if it doesn't exist.
3. **Always update `CLAUDE.md` with context-system instructions.** If no `CLAUDE.md` exists, create a minimal one with a project title placeholder. Then append the context-system block below — but only if the sentinel `<!-- context-system -->` is not already present.
4. **Install the session-start hook in `.claude/settings.json`.** The hook injects `state.md` into context when a session starts fresh or after `/clear`. Create `.claude/settings.json` with the JSON below if it doesn't exist. If it exists, merge the `SessionStart` entry into the existing JSON without disturbing other settings. If a `SessionStart` hook whose command mentions `state.md` is already present, skip it and say so. Do not ask — install and report.
5. **Do not scaffold anything else.** No tech stack, linting, CI, or project structure. This is purely the context layer.

## File contents

### `decisions.md`

```markdown
<!-- Append-only log of major decisions. Each entry: what was decided, when, why, and what was rejected. Versioned in git.

Entry format:

### YYYY-MM-DD — [Decision title]
**Status:** active
**Why:** [Rationale — what drove the decision]
**Rejected:** [What was considered and passed over, and why]

Status is `active` or `superseded`. An entry with no Status line is active. The Status line is the only part of an existing entry that may ever be edited. -->

# Decisions
```

### `state.md`

```markdown
<!-- Context bridge between sessions. Replace the contents of this file before ending a session so the next one can pick up without re-reading the entire codebase. -->

# State

## Where we ended
None

## What's working
None

## What's broken / in-progress
None

## Decided this session
None

## Next session should start with
None

## Landmines
None
```

### `scratch.md`

```markdown
<!-- Ephemeral working notes. Ideas, open questions, tangents during a session. Gitignored and wiped between sessions. -->

# Scratch
```

### Fallback `CLAUDE.md`

Only create this if no `CLAUDE.md` exists:

```markdown
# [Project Name]
```

### Context-system block for `CLAUDE.md`

Append this to `CLAUDE.md` if the sentinel `<!-- context-system -->` is not already present:

```markdown
<!-- context-system -->
## Context system

`state.md` is injected automatically at session start by a SessionStart hook — orient from it before acting. Run the orient skill for a full structured orientation with staleness checks.

Files:
- `state.md` — context bridge between sessions (tracked)
- `decisions.md` — append-only decision log (tracked)
- `scratch.md` — ephemeral working notes, wiped between sessions (gitignored)
```

### Session-start hook for `.claude/settings.json`

Merge this into `.claude/settings.json` (create the file and the `.claude` directory if needed). If the file already has a `hooks` or `SessionStart` key, merge this entry in alongside what's there:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear",
        "hooks": [
          {
            "type": "command",
            "command": "[ -f state.md ] && { echo 'state.md from the last preserved session:'; cat state.md; } || true"
          }
        ]
      }
    ]
  }
}
```

## Execution order

1. Use the existing files list above to determine which context files need to be created — skip any that already exist
2. If `.gitignore` exists, check whether `scratch.md` is already listed
3. Create missing files using the templates above
4. Append `scratch.md` to `.gitignore` if not already present — create `.gitignore` if it doesn't exist
5. Create fallback `CLAUDE.md` if needed
6. Append context-system block to `CLAUDE.md` if sentinel `<!-- context-system -->` not found
7. Install the session-start hook into `.claude/settings.json` — create or merge as needed, skip if a state.md hook is already present
8. Report what was created, what was skipped, and that the hook was installed
