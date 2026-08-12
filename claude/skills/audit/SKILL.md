---
name: audit
description: "Checks active decisions in decisions.md against the current codebase and git history, then reports which still hold, which have been violated, and which look superseded. Use when reviewing whether past decisions still hold, before a refactor, or when the decision log feels out of date."
allowed-tools: Read, Grep, Glob, Bash(git:*)
---

# Audit

Audit the decision log against reality. Read every active decision in `decisions.md`, look for concrete evidence in the code and git history, and report which ones still hold and which the project has drifted away from.

## Scaffold check

!`[ -f decisions.md ] && echo "DECISIONS_OK" || echo "DECISIONS_MISSING: run the scaffold skill first"`

**If the output above says DECISIONS_MISSING, stop immediately and tell the user to run the scaffold skill. Do not continue.**

## Recent git history

!`git log --oneline -20 2>/dev/null || echo "(no commits found)"`

Commits touching `decisions.md` (when the log itself was last maintained):

!`git log --oneline -5 -- decisions.md 2>/dev/null | grep . || echo "(decisions.md has never been committed)"`

## Rules

0. **This is read-only.** Search, read, and report. Do not fix code, do not rewrite decisions, do not start the work the audit implies. The single exception is rule 6 — flipping a Status line after the user confirms.

1. **Read `decisions.md` in full first.** Every entry, not a grep of the headings. You need the rationale and the rejected alternatives to know what evidence would confirm or contradict each one.

2. **Audit active entries only.** An entry marked `**Status:** superseded` is history — skip it silently. An entry with no Status line is active; audit it, and do not add the field.

3. **Look for evidence before judging.** For each active decision, search the codebase for what it would leave behind — the file, the import, the config key, the pattern it mandated or forbade — and check git history for commits that moved against it. Use `git log -S` for a rejected construct, `git log --oneline -- <path>` for a decision about a specific file.

4. **Cite or don't claim.** Every finding names a file path, symbol, or commit hash. "The retry decision holds" is not a finding; "`src/queue/retry.ts:41` still uses the documented exponential backoff" is.

5. **"Cannot verify" is a real finding, and absence of evidence is not violation.** If nothing in the code or history speaks to a decision — it was about process, about something not yet built, or about a system this repo doesn't contain — say you cannot verify it and say what would settle it. Never guess. Never infer a violation from silence.

6. **Propose Status flips; never apply them unasked.** A decision the project has deliberately moved past is a superseded candidate — say which later decision or commit replaced it, and ask whether to flip it. Only after the user confirms, edit that entry's `**Status:**` line to `superseded`. Change nothing else in the file: not the wording, not the date, not the order, and never a violated entry that the user hasn't decided to abandon.

7. **Violated and superseded are different findings.** Violated means the code contradicts a decision that is still supposed to hold — the drift is the bug. Superseded means the decision itself is obsolete. Don't quietly convert an inconvenient violation into a superseded candidate.

8. **Prose, not scores.** No percentages, no health grade, no compliance count. Group findings into the three buckets and let the evidence carry the weight.

## Output format

Present the audit as a single structured block. Omit any bucket that has no findings.

```
## Decision Audit

Audited [N] active decisions ([M] superseded entries skipped).

### Holding
- **[YYYY-MM-DD — Decision title]** — [what confirms it: file path, symbol, or commit]

### Violated
- **[YYYY-MM-DD — Decision title]** — [the specific contradiction: "rejected polling in favor of webhooks; commit a4f9c added a polling loop in src/sync.ts:88"]

### Superseded candidates
- **[YYYY-MM-DD — Decision title]** — [what replaced it, and the proposed flip to superseded]

### Cannot verify
- **[YYYY-MM-DD — Decision title]** — [why the evidence isn't in this repo, and what would settle it]
```

If there are superseded candidates, end with one line asking whether to flip them. Otherwise stop after the block — do not suggest fixes for the violations unless asked.

## Execution order

1. Check the scaffold output above — if `decisions.md` is missing, tell the user to run the scaffold skill and stop
2. Read `decisions.md` in full; list the active entries and count the superseded ones you're skipping
3. Read `CLAUDE.md` and `state.md` if they exist — they name the current shape of the project
4. For each active decision, decide what evidence would confirm or contradict it
5. Search for that evidence: Grep and Glob for code, `git log` / `git log -S` for history
6. Sort each decision into holding, violated, superseded candidate, or cannot verify — with a citation for each
7. Produce the audit block in the output format above
8. If any superseded candidates exist, ask whether to flip them; on confirmation edit only those `**Status:**` lines
