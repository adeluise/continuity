---
name: orient
description: "Start-of-session orientation that reads context files and recent git activity, then presents a structured summary. Use when beginning a session, after /clear, or when picking up a project cold."
allowed-tools: Read, Bash(git:*)
---

# Orient

Orient before acting. Read the project's context files and recent git activity, synthesize a structured summary, and present it. Do not start any work.

## Recent git activity

Recent commits:

!`git log --oneline -5 2>/dev/null || echo "(no commits found)"`

Last commit that touched `state.md` (the staleness anchor):

!`git log -1 --format="%h %ad %s" --date=short -- state.md 2>/dev/null | grep . || echo "(state.md has never been committed — staleness unknown)"`

Commits since `state.md` was last committed (the last preserved session boundary):

!`git log -1 --format=%H..HEAD -- state.md 2>/dev/null | git rev-list --stdin --oneline 2>/dev/null | grep . || echo "(none — state.md is current with git history, or the anchor above is unknown)"`

Working tree status:

!`git status --short 2>/dev/null | grep . || echo "(clean, or not a git repo)"`

## Rules

0. **This is read-only — do not start working.** The entire purpose of this skill is to orient. Read files, synthesize, present, and stop. Do not create, edit, or fix anything. Do not run tests. Do not write code. Wait for the user to direct next steps.

1. **Require the tracked scaffold files.** If `state.md` or `decisions.md` doesn't exist in the project root, stop and tell the user to run the scaffold skill first. Do not partially orient. A missing `scratch.md` is fine — it's gitignored, so it won't exist after a fresh clone. Treat it as empty and continue.

2. **Read everything before speaking.** Read `state.md`, `decisions.md`, `scratch.md`, and `CLAUDE.md` (if it exists) in full before producing any output. Combine them with the injected git context above to build the complete picture.

3. **Lead with what to do next.** The "Next session should start with" section of `state.md` is the most important signal. Put it first in the summary. The user wants to know the next action, not a recap.

4. **Be concrete, not narrative.** Use file paths, function names, error messages, and specific next steps. No "let's pick up where we left off" or "good progress was made." If `state.md` contains vague content, surface it as-is — don't embellish.

5. **Flag staleness.** The injected "commits since `state.md` was last committed" output is the staleness signal. If it lists commits, `state.md` predates them — say so explicitly and list those commits; the context files may be outdated. If the injected staleness anchor says `state.md` has never been committed, note that staleness can't be determined from git and ignore the commit list below it.

6. **Distrust the staleness check when `state.md` is uncommitted.** The anchor is the last commit that touched `state.md`. If the injected working tree status shows `state.md` as modified or untracked, say so and warn that the staleness result above is unreliable — the file on disk is newer than the anchor, so commits since the anchor may already be reflected in it.

7. **Flag emptiness.** If `state.md` sections are all "None" (the scaffold default), say so plainly: the context files exist but haven't been filled in yet. There's nothing to orient from — ask the user what they'd like to work on.

8. **Surface scratch.md only if non-empty.** Preserve wipes scratch.md clean. If it contains content, that's unusual — include it in the summary under a separate note. Otherwise omit it.

## Output format

Present the orientation as a single structured block:

```
## Session Orientation

### Next step
[Contents of "Next session should start with" from state.md — or "No next step recorded" if empty/None]

### Where we left off
[Contents of "Where we ended" from state.md]

### What's working
[Contents of "What's working" from state.md]

### What's broken / in-progress
[Contents of "What's broken / in-progress" from state.md — omit if None]

### Landmines
[Contents of "Landmines" from state.md — omit if None]

### Git since last session
[Summary of injected git context — commits since state.md was last committed, uncommitted changes. Omit if clean and nothing notable.]
```

After the block, print one line:

**Ready when you are. What should we work on?**

Do not say anything else after that line.

## Execution order

1. Check that `state.md` and `decisions.md` exist in the project root — if either is missing, tell the user to run the scaffold skill and stop
2. Read `state.md` in full
3. Read `decisions.md` in full
4. Read `scratch.md` in full (skip without error if it doesn't exist — it's gitignored)
5. Read `CLAUDE.md` if it exists (skip without error if it doesn't)
6. Combine file contents with the injected git context above
7. Check for staleness: use the injected "commits since state.md was last committed" output, and check the injected working tree status for an uncommitted `state.md` before trusting it
8. Produce the orientation block in the output format above
9. Stop. Wait for user direction.
