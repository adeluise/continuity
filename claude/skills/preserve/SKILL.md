---
name: preserve
description: "End-of-session context preservation that updates state.md, appends decisions to decisions.md, and wipes scratch.md. Use when wrapping up a session, before /clear, or when the user says they're done for now."
allowed-tools: Read, Edit, Write, Bash(git:*)
---

# Preserve

Preserve the current session's context into the three-file system so the next session can start cold. Run this at the end of a session, before `/clear`.

## Scaffold check

!`[ -f state.md ] && [ -f decisions.md ] && echo "SCAFFOLD_OK" || echo "SCAFFOLD_MISSING: run the scaffold skill first"`

**If the output above says SCAFFOLD_MISSING, stop immediately and tell the user to run the scaffold skill. Do not continue.**

## Session changes

Files changed this session:
!`git diff --stat`
!`git diff --cached --stat`

Changed file list:
!`git diff --name-only`
!`git diff --cached --name-only`

Commits made since `state.md` was last committed (work already committed this session — the diff above won't show it):

!`anchor=$(git log -1 --format=%H -- state.md 2>/dev/null); if [ -z "$anchor" ]; then git log --oneline -10 2>/dev/null || echo "(no commits)"; else git log --oneline "$anchor..HEAD" 2>/dev/null | grep . || echo "(none)"; fi`

## Rules

0. **Don't ask questions — just do it.** Derive everything from the session's conversation, the injected git diff above, and the current state of the three files. Present the result for review when done. The commit offer in rule 11 is the single deliberate exception.

1. **Require the tracked scaffold files.** If the scaffold check above says `SCAFFOLD_MISSING`, stop and tell the user to run the scaffold skill first. Do not create `state.md` or `decisions.md`. A missing `scratch.md` is fine — it's gitignored, so it won't exist after a fresh clone; the wipe step recreates it.

2. **Read before writing.** Before touching anything, read the current `state.md`, `decisions.md`, and `scratch.md` (if it exists) so you know what's already there.

3. **Use the injected context.** The git output above (uncommitted diff plus commits since the last preserve) was injected at skill load time. Combine it with the conversation history — don't rely on either source alone; the diff misses work already committed this session, and the commit list covers it. Do not re-run git unless the injected output is empty.

4. **Be concrete, not reflective.** Every section in `state.md` should contain specific file paths, function names, error messages, or next steps. Never write vague summaries like "made good progress" or "things are working well."

5. **Decisions have a threshold.** Only append to `decisions.md` if a decision meets at least two of these criteria:
   - Closes a door — choosing A over B, and going back would cost real time
   - Future-you would ask "why did we do it this way?" and the answer isn't obvious from the code
   - Something was explicitly rejected
   - Changes the project's direction or constraints

   If nothing qualifies, don't append anything. Don't log variable naming, formatting, or anything reversible in five minutes.

6. **Every new decision is `**Status:** active`.** It's the first field line of the entry. An existing entry with no Status line is active — do not backfill it.

7. **`decisions.md` is append-only, with one exception: the Status line.** Add new entries after the last existing entry. Never remove, reword, or reorder an existing entry. If a decision made this session *directly reverses* an earlier entry, change that entry's Status from `active` to `superseded` — that one line, nothing else — and say which entry you flipped and why. Merely revisiting or extending a decision is not a reversal.

8. **Harvest `scratch.md` before wiping it.** Read it and move anything still relevant into the matching existing `state.md` section — open questions and next actions into "Next session should start with", gotchas into "Landmines", unfinished threads into "What's broken / in-progress". Do not add a new section for them. Then list for the user, explicitly, every scratch note you discarded, so nothing disappears silently.

9. **Wipe scratch.md clean.** Only after the harvest. Replace its contents with just the header comment and heading — nothing else. Create it if it doesn't exist.

10. **Present the result.** After writing all three files, show the user what was written to `state.md`, what was appended to `decisions.md` (if anything), any Status flip, and the discarded scratch notes — so they can correct it before the session ends.

11. **Offer to commit.** After presenting, ask once: commit `state.md` and `decisions.md`? A single yes/no — do not negotiate the message or stage anything else. On yes, `git add state.md decisions.md` and commit with a short lowercase-style message like `Preserve session state`. On no, say nothing further. This is the only question this skill asks; the offer is what makes committing the user's call.

## Templates

### `state.md` — full replacement

```markdown
<!-- Context bridge between sessions. Replace the contents of this file before ending a session so the next one can pick up without re-reading the entire codebase. -->

# State

## Where we ended
[What was the last thing being worked on. File paths, function names, specific context.]

## What's working
[Features, systems, or components confirmed working this session.]

## What's broken / in-progress
[Anything left incomplete, failing, or partially implemented. Include error messages if relevant.]

## Decided this session
[Quick summary of decisions made — details go in decisions.md.]

## Next session should start with
[The first concrete action for next session. Not a wish list — the single most important next step, then supporting items.]

## Landmines
[Gotchas, surprising behavior, things that look wrong but are intentional, or things that look right but are broken.]
```

### `decisions.md` — append only

```markdown
### YYYY-MM-DD — [Decision title]
**Status:** active
**Why:** [Rationale — what drove the decision]
**Rejected:** [What was considered and passed over, and why]
```

### `scratch.md` — wiped

```markdown
<!-- Ephemeral working notes. Ideas, open questions, tangents during a session. Gitignored and wiped between sessions. -->

# Scratch
```

## Execution order

1. Check that `state.md` and `decisions.md` exist in the project root — if either is missing, tell the user to run the scaffold skill and stop
2. Read current contents of `state.md`, `decisions.md`, and `scratch.md` (if it exists)
3. Decide which `scratch.md` notes are still relevant and which section of `state.md` each belongs in
4. Overwrite `state.md` with filled-in template using the injected git context (diff plus commits), conversation history, and the harvested scratch notes
5. Review session for decisions that meet the threshold — append to `decisions.md` with `**Status:** active` if any qualify, skip if none do
6. If a new decision directly reverses an earlier entry, flip that entry's Status line to `superseded`
7. Wipe `scratch.md` back to its empty template (create it if missing)
8. Show the user what was written to `state.md`, what was appended to `decisions.md`, any Status flip, and every discarded scratch note
9. Offer once to commit `state.md` and `decisions.md`; commit only on yes
