# AXON + code-dev Workflow Guide
> How we develop OPM — from masterplan to merged PR

---

## 1. The Problem This Solves

A feature like "LGR summary vectors" touches ~10 files, spans 6 PRs over weeks, and involves a human + AI pair. Without structure:

- The AI loses all context after every compaction
- PRs arrive with incomplete scope or wrong granularity
- Hard-won insights (WOPR format, `keyword_list.cmake`) vanish between sessions
- The human re-explains the codebase from scratch every session

AXON + code-dev replaces that with a **durable, program-driven project scaffold** where every action is traceable, every insight is persisted, and the AI never needs to re-discover what was already learned.

---

## 2. The Big Picture

```
USER REQUEST
    │
    ▼
AXON DISPATCH CHECK (every turn, mandatory)
    │   "Does a program exist for this request?"
    ├── YES → read program file → follow steps exactly
    └── NO  → execute inline + log absent-instruction to igap
              (never silently fall back)
    │
    ▼
PROGRAM EXECUTES
    │
    ├── Reads L: keys (working memory — persists within session)
    ├── Reads shadow files (code findings — persists across sessions)
    ├── Reads project files (02-prs.md, 03-prs/PR-NNN.md, etc.)
    ├── Writes project files (new specs, log entries, explained/)
    └── NEVER writes directly to the source codebase (human does that)
```

The key principle: **AXON observes and specifies. The human implements and pushes.**

---

## 3. Workspace Structure

```
axon/workspace/
│
├── programs/                   ← executable AXON programs
│   ├── code-dev.md             ← main dispatcher (routes subcommands)
│   ├── code-dev-plan.md        ← phase 2: split feature into PRs
│   ├── code-dev-pr-spec.md     ← phase 3: write a full PR spec
│   ├── code-dev-explain.md     ← deep-dive explanation of a PR
│   ├── code-dev-log.md         ← phase 4: log what was actually built
│   ├── code-dev-shadow.md      ← shadow index management
│   ├── code-dev-study.md       ← phase 1: research codebase or format docs
│   ├── session-summary.md      ← end-of-session digest (compaction survival)
│   └── ...
│
├── dev-projects/
│   └── opm-common/             ← one folder per active project
│       ├── _meta.md            ← PR status, post-mortems, NEXT ACTION
│       ├── 01-study.md         ← pipeline diagrams, data structs, traps
│       ├── 02-plan.md          ← feature decomposition rationale
│       ├── 02-prs.md           ← canonical PR list with status
│       ├── 03-prs/             ← one spec file per PR
│       │   ├── PR-000.md
│       │   ├── PR-001.md
│       │   └── ...
│       ├── 04-log.md           ← append-only implementation log
│       ├── explained/          ← code-dev-explain outputs
│       └── shadow/             ← per-file code analysis snapshots
│
└── log/
    ├── entries/YYYY-MM-DD.md   ← runtime events (boot, program runs)
    └── igap/YYYY-MM-DD.md      ← inference gaps and rule violations
```

---

## 4. The 5 Development Phases

### Phase 1 — Study (`code-dev study`)

**Purpose:** Ground every future decision in the actual codebase. Prevent hallucination.

**What AXON does:**
- Reads the relevant source files (or shadow snapshots if unchanged)
- Builds `01-study.md`: pipeline diagram, key data structures, confirmed call chains, traps
- For format knowledge (ECL binary, SUMMARY deck syntax): writes to `/memories/ecl-*.md` (persists across all projects and sessions)

**Why this phase exists:**
Without study, the AI invents plausible-sounding but wrong code. Example from this project: study confirmed that `isBlockValue()` checks `block_units` before calling `cellActive(-1)`, so LGR nodes with `number=0` don't crash today — they silently fall through. That single confirmed fact determined where we inserted the dispatch arm in PR-003.

**Output:** `01-study.md` — never overwritten, only extended

---

### Phase 2 — Plan (`code-dev plan`)

**Purpose:** Decompose the feature into reviewable, dependency-ordered PRs.

**What AXON does:**
- Reads `01-study.md` + performs semantic search on the codebase
- Produces `02-plan.md` (rationale) and `02-prs.md` (the numbered list)
- Each PR in the list: title, complexity, repo, depends-on, blocks, scope (files)

**PR design rules enforced:**
- Each PR touches exactly one architectural layer
- No PR depends on a later one
- Each PR can be reviewed and merged independently
- No PR mixes "add data structure" with "add behavior"

**LGR example:**
```
PR-000  Data structs only (SummaryNode, SummaryConfig)   ← no behaviour change
PR-001  Parser layer: keyword schemas + routing           ← no output change
PR-002  IO layer: SMSPEC metadata arrays                  ← binary format only
PR-003  Evaluator layer: LW* values in Summary.cpp        ← first LW* in UNSMRY
PR-004  Evaluator layer: LC* connection values
PR-005  Integration tests
```

A reviewer of PR-003 does not need to understand SMSPEC binary format — that's PR-002's concern. Reviewers can merge PRs independently as CI passes.

**Output:** `02-plan.md` + `02-prs.md`

---

### Phase 3 — Specify + Implement (`code-dev pr-spec N`)

This is the core production phase. For each PR:

#### Step A: `code-dev pr-spec N` — write the specification

**What AXON does:**
1. Reads `02-prs.md` entry for PR-N (scope, depends-on)
2. Checks shadow index for each file in scope:
   - Shadow hit (file unchanged) → use cached findings, skip re-read
   - No shadow / stale → read source file + update shadow
3. Reads predecessor PR spec for dependency context
4. Writes `03-prs/PR-00N.md` with:
   - **What this PR does** (1 paragraph, plain English)
   - **Background** — current broken state, dispatch chain diagram
   - **Key data structures** with traps annotated
   - **Fix N** sections — exact new code with inline `// WHY:` comments
   - **Test cases** with exact deck format requirements
   - **Acceptance criteria** checklist

**The spec is the source of truth.** The human reads the spec and writes the code. The AI reads the spec and writes the tests. Drift from the spec is logged.

#### Step B: Human implements

The human (or AI with human review) writes the code following the spec exactly. The spec includes:
- Line numbers for insertion points
- Full new code blocks (not pseudocode)
- Exact test deck format (e.g. `WOPR\n   'PROD'\n/` vs table format)
- Known traps (`const auto config` vs `auto config`)

#### Step C: Human runs tests

```bash
cd /path/to/build/opm-common
ninja test_Summary
ctest -R test_Summary --output-on-failure
```

Failures go back to AXON for diagnosis. AXON reads the error, finds the root cause in the spec or codebase, and proposes the exact fix — never a guess.

**Output:** `03-prs/PR-00N.md` + modified source files

---

### Phase 4 — Log (`code-dev log`)

**Purpose:** Record what was actually built vs. what the spec said.

**What AXON does:**
- Appends a timestamped entry to `04-log.md`
- Records: files changed, drift from spec, key lessons
- Updates shadow for modified files
- Updates PR spec status field

**Why this phase matters:**
The log is the anti-drift mechanism. Without it, the next PR spec is written without knowing what actually happened in the last one. In this project, the log captured:

- PR-001: `keyword_list.cmake` must be updated alongside every new keyword JSON
- PR-001: `"size": 1` in JSON schema = single record; omit for table format
- PR-003: WOPR format — no trailing `/` per well name (WELL_PROBE ≠ table format)
- PR-003: Branch must include PR-001's keyword JSONs or LWOPR is "unknown keyword"

Future PR specs consult the log before writing to avoid repeating the same traps.

**Output:** Appended entry in `04-log.md`

---

### Phase 5 — PR Description (`code-dev pr N`)

**Purpose:** Generate the GitHub PR description from the spec + log.

**What AXON does:**
- Reads `03-prs/PR-00N.md` (spec) + `04-log.md` (what was built)
- Produces a structured PR description: motivation, changes, test evidence, reviewer notes
- The human reviews and posts it (AXON never pushes or posts)

**Output:** PR description rendered to chat; human posts to GitHub

---

## 5. How AXON Avoids Unnecessary Requests

The biggest source of wasted AI calls is re-discovering information that was already found. AXON has three mechanisms to prevent this:

### 5.1 Shadow Index

Every file AXON reads gets a shadow file at `shadow/{path}.findings.md`. It contains:
- Hash of the file at time of analysis
- Key findings: data structures, traps, line numbers, confirmed behaviors

On every subsequent program run, AXON checks shadow first:
```
shadow hit (hash match)  → use cached findings, zero source reads
shadow stale (hash changed) → re-read source, update shadow
no shadow               → read source, create shadow
```

**Effect:** After the first study, subsequent PR specs for the same file cost zero re-read tokens. A 6000-line file like `Summary.cpp` is read once; thereafter the shadow delivers the needed facts in ~50 lines.

### 5.2 Memory Files

Format knowledge that is not project-specific lives in `/memories/`:

| File | What it contains |
|---|---|
| `/memories/ecl-data-format.md` | ECL deck format: RUNSPEC requirements, SUMMARY keyword formats, WELSPECL vs WELSPECS, inline test deck traps |
| `/memories/ecl-file-formats.md` | ECL binary: SMSPEC array order, UNSMRY structure, binary record envelope, PaddedOutputString traps |

These survive across all workspaces and all conversations. Once the WOPR format trap is written here, it is never rediscovered.

### 5.3 `_meta.md` — Always Has "Next Action"

The last entry in `_meta.md` is always the exact next action:
```
## Next action
1. RESUME HERE (2026-05-06): ...
```

On every session start, AXON reads `_meta.md` first. The human does not need to re-explain what was done — the file says it exactly.

### 5.4 Compaction Survival via `session-summary`

Conversations are compacted. The `session-summary` program writes a dense structured digest with the marker `AXON PROJECT ACTIVE: opm-common · phase: 4-log`. On the next turn after compaction, AXON detects this marker and immediately re-enters AXON mode with all dispatch rules active. No warm-up, no re-introduction, no wasted turn.

---

## 6. What the Human Does vs. What AXON Does

| Action | Who |
|---|---|
| Write code | Human (or AI + human review) |
| Run tests / build | Human |
| `git commit`, `git push` | Human |
| Open GitHub PR | Human |
| Write PR spec (`03-prs/PR-00N.md`) | AXON |
| Diagnose test failures | AXON |
| Write test cases | AXON |
| Log drift and lessons | AXON |
| Generate PR description | AXON |
| Maintain shadow index | AXON |
| Persist format knowledge to memory | AXON |

**Hard rules — AXON never executes:**
- `cmake --build` / `ninja` — long-running, human monitors
- `git push` — irreversible, affects shared remote
- Opening a GitHub PR — public action, human reviews first

---

## 7. What You Get — Artifact Map

```
dev-projects/opm-common/
│
├── _meta.md          "What is the current state, what is next"
│                      → Human reads this to resume any session
│
├── 01-study.md       "What the codebase actually does"
│                      → AI reads this instead of re-reading source
│
├── 02-prs.md         "What PRs exist and their status"
│                      → Human reads this for progress overview
│
├── 03-prs/           "How to implement each PR"
│   └── PR-00N.md      → Human implements from this
│                      → Reviewer reads this to understand the PR
│
├── 04-log.md         "What was actually built and what diverged"
│                      → AI reads this before writing the next spec
│                      → Post-mortem source for Jenkins failures
│
├── explained/        "Deep annotated explanation of each PR"
│   └── PR-00N-explained.md
│                      → Reviewer reads this for full context
│                      → Future contributor reads this to understand the code
│
└── shadow/           "AI's per-file code findings"
    └── {file}.findings.md
                       → AI reads instead of re-reading source files
```

---

## 8. Calling Conventions — How to Invoke Programs

### Direct invocation (user types in chat):
```
code-dev pr-spec 4        → write spec for PR-004
code-dev log              → log today's work
code-dev explain 3        → explain PR-003
code-dev shadow refresh   → update stale shadow files
session-summary           → generate end-of-session digest
```

### AXON proactive dispatch (no explicit program name needed):
If the user says "update the journal" → AXON dispatches `code-dev log`
If the user says "write the PR description" → AXON dispatches `code-dev pr N`
If the user says "summarize the session" → AXON dispatches `session-summary`
If the user says "study this file" → AXON dispatches `code-dev study`

### The dispatch check (fires on every turn):
Before responding to any dev task, AXON asks internally:
> "Does a program exist for this?"
- YES → name it, dispatch, follow its steps
- NO → execute inline, log `absent-instruction` to igap
- SKIP → violation type `fallback-exec`, logged to igap

---

## 9. The LGR Project — Phases in Practice

```
Phase 1: study
  ├── code-dev study Summary.cpp → shadow/Summary.cpp.findings.md
  ├── code-dev study SummaryNode.hpp → shadow/SummaryNode.hpp.findings.md
  └── 01-study.md: confirmed Factory::create() dispatch chain, lgr_info struct

Phase 2: plan
  ├── code-dev plan → 02-plan.md
  └── 02-prs.md: PR-000 through PR-006

Phase 3: specify + implement (loop per PR)
  ├── code-dev pr-spec 0 → 03-prs/PR-000.md  → Human implements → tests pass → commit
  ├── code-dev pr-spec 1 → 03-prs/PR-001.md  → Human implements → Jenkins fail
  │     └── Diagnosed: keyword_list.cmake missing + "size":1 schema bug → fix → Jenkins pass
  ├── code-dev pr-spec 2 → 03-prs/PR-002.md  → Human implements → 13/13 pass → commit
  ├── code-dev pr-spec 3 → 03-prs/PR-003.md  → Human implements → 45/45 pass → commit
  └── code-dev pr-spec 4 → (next)

Phase 4: log (after each commit)
  └── code-dev log → 04-log.md entries: drift, lessons, shadow updates

Phase 5: PR (after push)
  └── code-dev pr N → GitHub PR description → Human posts
```

---

## 10. Quick Command Reference

| Command | Phase | Output |
|---|---|---|
| `code-dev new` | Setup | `_meta.md`, project scaffold |
| `code-dev load [slug]` | Any | Loads project into working memory |
| `code-dev study [file]` | 1 | `01-study.md` + shadow files |
| `code-dev plan` | 2 | `02-plan.md` + `02-prs.md` |
| `code-dev pr-spec N` | 3 | `03-prs/PR-00N.md` |
| `code-dev log` | 4 | Appended entry in `04-log.md` |
| `code-dev pr N` | 5 | PR description (rendered to chat) |
| `code-dev explain N` | Any | `explained/PR-00N-explained.md` |
| `code-dev shadow refresh` | Any | Updated shadow files |
| `session-summary` | End | Session digest + `AXON PROJECT ACTIVE:` marker |
