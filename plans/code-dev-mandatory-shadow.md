# PLAN: code-dev-mandatory-shadow
Goal:    Enforce mandatory shadow-file coverage for every file read or modified in a code-dev session
Created: 2026-05-07
Updated: 2026-05-07
Status:  active

---

## MANDATORY RULE

**Shadow is a hard gate — not a suggestion. It applies to every project, every codebase, every session.**

### Rule 1 — Shadow before source (ALWAYS)
Before reading any source file, check shadow first.
Shadow hit → load shadow, skip source read entirely.
Shadow miss or stale → read source AND write shadow immediately after.
**No exceptions. No "quick glances". No "I'll shadow it later".**

### Rule 2 — Shadow gates every code-dev entry
When any code-dev subcommand (study, plan, pr, log, audit) fires against a project with a codebase:
- The shadow index stats are shown immediately
- Stale files are flagged before any work begins
- Questions and research start from shadow findings, not raw source

### Rule 3 — Questions leverage shadow first
When a question is asked about any code — function, struct, behavior, API — the answer must:
1. Check shadow for prior findings on that file
2. If shadow covers it → answer from shadow (zero source tokens)
3. If shadow is absent or does not cover it → read source → write shadow → then answer

**Going directly to the codebase without checking shadow is a violation.**
Violation type: `shadow-bypass` — LOG(WARN, "shadow-bypass: {file}") + append igap entry.

### Rule 4 — Every modified file is shadowed before session ends
All edits must have a shadow update logged before the session closes.
This is enforced by the P4 session-end gate.

This applies to:
- Source files read for context (even once, even briefly)
- Headers or schemas read as dependencies
- Any file edited, regardless of how small the change
- Any file mentioned in a code-dev question or discussion

---

## SHADOW FILE LOCATION

```
/home/arturcastiel/opm-hd/axon-project/axon/my-axon/dev-projects/<project>/shadow/<absolute-path>.findings.md
```

Example for a source file at `/home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp`:

```
shadow/home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp.findings.md
```

---

## SHADOW FILE TEMPLATE

```markdown
---
name: <filename only, e.g. SummaryConfig.cpp>
description: <one-line purpose of this file>
type: findings
---

## Summary

<2–4 sentences: what this file is, what subsystem it belongs to, what it is responsible for>

## Key Structures

| Name | Kind | Purpose |
|------|------|---------|
| <StructName> | struct/class/enum/fn | <what it does> |

## Findings Log

| Date | Context | Finding |
|------|---------|---------|
| YYYY-MM-DD | <PR or task> | <concrete observation — line numbers, API names, behavior, stale patterns, edge cases> |
```

Rules for the findings log:
- Each entry must be a concrete observation, not a vague note
- Include line numbers or function names where relevant
- One row per distinct finding — do not bundle multiple observations into one cell
- New findings are appended as new rows; do not overwrite old rows

---

## PHASES

---

### Phase 1 — PRE-SESSION CHECK
*Entry: session is starting, project directory known*

- [ ] P1-1: Identify the active dev-project path (e.g. `dev-projects/opm-common`)
- [ ] P1-2: List all files touched in this branch vs base: `git diff <base> --name-only`
- [ ] P1-3: For each touched file, check whether a shadow exists:
  ```bash
  # Quick-check: list touched files with no shadow counterpart
  git diff <base> --name-only | while read f; do
    shadow="dev-projects/<project>/shadow/${f}.findings.md"
    [ -f "$shadow" ] || echo "UNSHADOWED: $f"
  done
  ```
- [ ] P1-4: Record the unshadowed file list — this is the shadow debt for this session
- [ ] P1-5: Note which files have shadows already — read those shadows before touching the files

**Exit gate:** shadow debt list is written down. Existing shadows are loaded before reading their files.

---

### Phase 2 — PER-FILE RULE (during session)
*Entry: about to read or edit any file*

**Before reading a file:**
- [ ] P2-1: Check if a shadow exists for this file
- [ ] P2-2: If a shadow exists — read it first to load prior findings before reading the source

**After reading a file (before moving to the next file):**
- [ ] P2-3: If no shadow exists — create it now using the template above
- [ ] P2-4: If a shadow exists — append any new findings discovered during this read
- [ ] P2-5: Confirm the shadow file exists and is non-empty before closing the file

**Exit gate (per file):** shadow file exists, summary is accurate, findings log has at least one entry from this session.

---

### Phase 3 — POST-EDIT UPDATE RULE
*Entry: a file has just been edited*

- [ ] P3-1: Immediately after saving an edit — open the shadow file for that source file
- [ ] P3-2: Append a findings log entry for today's date describing what was changed and why
- [ ] P3-3: Update the Key Structures table if new structs, functions, or enums were added or renamed
- [ ] P3-4: Update the Summary section if the file's role or responsibility changed
- [ ] P3-5: Do NOT move on to the next file until this shadow update is complete

**Exit gate:** shadow reflects the current state of the file. The edit is described in the findings log.

---

### Phase 4 — SESSION-END GATE
*Entry: session work is complete, about to close*

- [ ] P4-1: Collect the full list of files touched this session (read + edited)
- [ ] P4-2: Run the quick-check command (see below) to find any unshadowed files
- [ ] P4-3: For every file flagged as unshadowed — create or update its shadow before ending
- [ ] P4-4: Verify no shadow file is empty or contains only the template skeleton with no findings
- [ ] P4-5: Commit shadow files as part of the session commit or as a dedicated shadow-update commit

**Exit gate:** zero unshadowed files in the touched set. All shadow files have at least one dated finding entry from this session.

---

### Phase 5 — COVERAGE GROWTH (across sessions)
*Entry: ongoing — applies between sessions*

- [ ] P5-1: Shadow files accumulate organically — each session adds coverage for files read in that session
- [ ] P5-2: Do not attempt to pre-shadow the entire codebase upfront — coverage grows with actual work
- [ ] P5-3: At the start of each session (P1), check what is already shadowed and read those shadows
- [ ] P5-4: Over multiple sessions, the shadow directory becomes a persistent knowledge base of studied files
- [ ] P5-5: If a file was shadowed in a prior session and is re-read, always append new findings — never discard old ones

**Exit gate (per session):** net coverage is higher at end of session than at start.

---

## ANTI-PATTERNS

| Anti-pattern | Why it is wrong | Correct approach |
|---|---|---|
| Batching shadow updates ("I'll write them all at the end") | Findings are forgotten; the batch never happens before session closes | Update shadow immediately after each file — one file, one shadow, move on |
| Skipping shadows for "small reads" | Every read adds knowledge; a skipped shadow means a blind next session | No size threshold. Read = shadow. Always. |
| Creating an empty shadow (template only, no findings) | An empty shadow is worse than no shadow — it signals coverage where there is none | Do not create the file until you have at least one real finding to write |
| Writing the shadow before the edit ("pre-emptive update") | Shadow must reflect what is in the file now, not what you plan to do | Write or update shadow after the edit, never before |
| Updating Summary and Key Structures but not the findings log | The log is the audit trail — summary without a dated entry is not traceable | Always add a dated row to the findings log, even for minor reads |
| Shadowing only edited files, not read-only files | Files read for context shape decisions — their findings matter | Any file opened and studied gets a shadow, regardless of whether it was edited |

---

## QUICK-CHECK COMMANDS

```bash
# List all files touched in this branch (vs upstream master) with no shadow
PROJECT_DIR="/home/arturcastiel/opm-hd/axon-project/axon/my-axon/dev-projects/<project>"
git -C /home/arturcastiel/opm-hd/projects/opm-development/opm-common \
  diff upstream/master --name-only | while read f; do
    abs="/home/arturcastiel/opm-hd/projects/opm-development/opm-common/$f"
    shadow="${PROJECT_DIR}/shadow${abs}.findings.md"
    [ -f "$shadow" ] || echo "UNSHADOWED: $f"
done

# Count total shadow files in project
find "${PROJECT_DIR}/shadow" -name "*.findings.md" | wc -l

# List all shadow files with their last-modified date
find "${PROJECT_DIR}/shadow" -name "*.findings.md" \
  -printf "%TY-%Tm-%Td  %p\n" | sort
```

---

## SESSION CHECKLIST (copy per session)

```
Shadow compliance checklist — session: <date> — project: <project>
Branch: <branch>

[ ] P1 — Pre-session check done; shadow debt list written
[ ] P1 — Existing shadows loaded before reading their files

During session (one row per file touched):
  File: <path>  Read: [ ]  Edited: [ ]  Shadow created/updated: [ ]
  File: <path>  Read: [ ]  Edited: [ ]  Shadow created/updated: [ ]
  ...

[ ] P4 — Quick-check run; zero unshadowed files flagged
[ ] P4 — All shadow files have at least one dated entry from today
[ ] P4 — Shadow files committed (or staged for commit)
```
