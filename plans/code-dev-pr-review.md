# PLAN: code-dev-pr-review
Goal:    General-purpose workflow for reviewing, harmonizing, and submitting PRs in code-dev mode
Created: 2026-05-07
Updated: 2026-05-07
Status:  active

---

## OVERVIEW

A repeatable 9-phase process for taking a branch from "needs review" or
"diverged from dependency" to "clean, tested, documented, and ready to push".
Distilled from the PR-000 (external review response) and PR-001 (API
harmonization) workflows on opm-common.

Each phase has a clear entry condition, a set of tasks, and an exit gate —
a verifiable condition that must be true before moving to the next phase.

---

## PHASES

---

### Phase 1 — CONTEXT LOAD
*Entry: branch exists, work is about to begin*

- [ ] P1-1: Read project tracking (`02-prs.md`) — identify PR status, dependencies, blockers
- [ ] P1-2: Read implementation log (`04-log.md`) — understand what changed and why since last session
- [ ] P1-3: Identify upstream state — has any dependency PR been merged? Run `git fetch --all`, check `git log upstream/master`
- [ ] P1-4: Identify divergence — run `git log <branch>..upstream/master` and `git log upstream/master..<branch>` to quantify drift
- [ ] P1-5: List every file the PR touches (`git diff upstream/master --name-only`)

**Exit gate:** clear picture of what the branch adds, what it depends on, and what (if anything) has changed underneath it.

---

### Phase 2 — STUDY (shadow phase)
*Entry: file list from Phase 1 known*

- [ ] P2-1: For each file the PR touches — read it in full and write/update a shadow findings file
- [ ] P2-2: For each dependency file (files the PR *reads from* but does not own) — read and shadow
- [ ] P2-3: In each shadow file, record: key structures, line numbers of interest, API used, known stale patterns
- [ ] P2-4: If a dependency PR was revised upstream — read the new version of its files; note the delta in the shadow files
- [ ] P2-5: Build a conflict map table: `file | region | status (keep / update / drop) | reason`

**Exit gate:** every touched file has a shadow findings log. Conflict map is complete with no "unknown" entries.

---

### Phase 3 — CONFLICT ANALYSIS
*Entry: shadow files complete, conflict map drafted*

- [ ] P3-1: Identify API drift — old call site patterns vs new API signatures (document exact before/after)
- [ ] P3-2: Identify superseded work — code added by this branch that is now duplicated by an upstream merge (mark as DROP)
- [ ] P3-3: Identify design decisions that need revisiting — e.g., fields or coordinates that belong at a different layer
- [ ] P3-4: For each conflict: determine resolution strategy and note it in the conflict map
- [ ] P3-5: Identify verification pattern for each fix — what grep, what build target, what test will confirm the fix is complete

**Exit gate:** every conflict has a resolution strategy and a verification action. No ambiguous entries.

---

### Phase 4 — HARMONIZATION PLAN
*Entry: conflict analysis complete*

- [ ] P4-1: Write numbered steps (not vague phases — specific, actionable changes)
- [ ] P4-2: Each step must state: what file, what region, what change, what verification
- [ ] P4-3: Order steps: rebase first, API fixes before test fixes, implementation before tests
- [ ] P4-4: Add a build + test step as the final numbered item
- [ ] P4-5: Save plan to `03-prs/PR-XXX-HARMONIZATION.md`

**Exit gate:** plan is written, numbered, reviewable. Each step is independently executable.

---

### Phase 5 — REBASE
*Entry: harmonization plan written*

- [ ] P5-1: Backup current branch — `git branch <branch>-bak`
- [ ] P5-2: Fetch upstream — `git fetch --all`
- [ ] P5-3: Rebase branch onto new base — `git rebase upstream/master`
- [ ] P5-4: At each conflict: apply conflict map decision (keep upstream / keep ours / manual merge)
- [ ] P5-5: For superseded commits (old dependency PR commit now in upstream) — use `git rebase --skip`
- [ ] P5-6: Verify rebase result — `git log --oneline -10`, check that superseded commits are gone

**Exit gate:** branch rebased cleanly onto new base. No orphaned dependency commits in history. `git status` clean.

---

### Phase 6 — EXECUTION (apply fixes)
*Entry: rebase complete*

- [ ] P6-1: Work through harmonization plan steps in order
- [ ] P6-2: After each file edit — run targeted grep sweep: `grep -c "<stale_pattern>" <file>` must return 0
- [ ] P6-3: Update shadow findings file for each file after editing it
- [ ] P6-4: Do not batch fixes across multiple files before verifying — fix one region, verify, move on

**Exit gate:** all harmonization steps applied. Zero stale pattern matches across all modified files. All shadow files updated.

---

### Phase 7 — VERIFICATION
*Entry: all fixes applied*

- [ ] P7-1: Final grep sweep across ALL modified files for any stale API patterns
- [ ] P7-2: Build — `cmake --build <build-dir> --target <target> -- -j$(nproc)`
- [ ] P7-3: Run targeted ctest — `ctest -R "^(<test1>|<test2>|...)$" --output-on-failure`
- [ ] P7-4: Confirm 100% pass rate — no failures, no skips
- [ ] P7-5: Note test targets and times for PR documentation

**Exit gate:** build clean, all relevant tests passing.

---

### Phase 8 — COMMIT ORGANIZATION
*Entry: verification green*

- [ ] P8-1: Backup current branch state if not already done — `git branch <branch>-bak`
- [ ] P8-2: Identify logical groupings — typically: schemas → implementation → tests
- [ ] P8-3: `git reset --mixed <base>` to unstage all changes (NOT soft — mixed empties the index)
- [ ] P8-4: Stage and commit each logical group separately
- [ ] P8-5: Commit message format: `<scope>: <what> — <why>` with a body explaining non-obvious decisions
- [ ] P8-6: Each commit must be independently reviewable — no commit should break the build
- [ ] P8-7: Verify final log — `git log --oneline -8`, confirm clean linear history

**Exit gate:** clean commit history with 2–4 logical commits. Each commit title and body accurate and complete.

---

### Phase 9 — DOCUMENTATION + TRACKING
*Entry: commits organized*

- [ ] P9-1: Write GitHub PR description to `03-prs/PR-XXX-github-description.md`:
  - Title (≤70 chars)
  - One-paragraph summary
  - Depends-on (PR numbers, commit hashes)
  - Commits table (hash + description)
  - Per-section changes (schemas / implementation / tests)
  - Testing section (exact ctest command + result)

- [ ] P9-2: Write technical explanation to `03-prs/PR-XXX-explain.md`:
  - Context and motivation
  - Architecture diagram if relevant (two-layer, pipeline, etc.)
  - Design decisions with rationale (why X was done this way, not Y)
  - Implementation details (key functions, filtering logic, edge cases)
  - Explicit "what this PR does NOT do" section (scope boundaries)

- [ ] P9-3: Update `02-prs.md` — set PR status to `✅ ready to push` with correct commit hash
- [ ] P9-4: Add log entry to `04-log.md` — date, branch, what changed, files affected, pending items
- [ ] P9-5: Update AXON memory if any non-obvious lessons emerged (feedback, project, reference types)

**Exit gate:** both PR documents written. Project tracking files updated. Shadow files current.

---

## ANTI-PATTERNS TO AVOID

| Anti-pattern | Why | Correct approach |
|---|---|---|
| `git reset --soft` then selective `git add` | soft keeps index full — commits pick up everything | Use `git reset --mixed` to empty the index first |
| Fix multiple files before running grep verification | stale calls compound silently | Fix one region, grep to zero, move on |
| Amending commits with pre-commit hook failures | amend modifies PREVIOUS commit, losing work | Fix issue, re-stage, create NEW commit |
| Merging upstream into branch | rewrites shared history, confuses reviewers | Always rebase, never merge |
| Skipping shadow file updates after editing | shadow goes stale, next session starts blind | Update shadow immediately after each file edit |
| Writing PR description from memory | drifts from actual code | Generate from `git diff --stat` + reading final file state |

---

## QUICK REFERENCE — KEY COMMANDS

```bash
# Branch state
git log --oneline -10
git diff upstream/master --name-only
git diff upstream/master --stat

# Rebase
git fetch --all
git branch <branch>-bak              # backup first
git rebase upstream/master
git rebase --skip                    # drop superseded commit

# Verification grep
grep -c "<stale_pattern>" <file>     # must return 0

# Commit organization
git reset --mixed upstream/master    # empty index, keep working tree
git add <file1> <file2>
git commit -m "$(cat <<'EOF'
scope: what this commit does

Why it does it — the non-obvious constraint or decision.
EOF
)"

# Test
cmake --build <build-dir> --target <target> -- -j$(nproc)
ctest --test-dir <build-dir> -R "^(<suite1>|<suite2>)$" --output-on-failure
```

---

## TEMPLATE CHECKLIST (copy per PR)

```
PR-XXX review checklist
Branch: <branch>
Base:   <upstream/master or dependency branch>

[ ] P1 — Context loaded, divergence quantified
[ ] P2 — All files shadowed, conflict map drafted
[ ] P3 — All conflicts resolved in plan
[ ] P4 — Harmonization plan written to 03-prs/
[ ] P5 — Rebase complete, backup branch exists
[ ] P6 — All fixes applied, zero stale grep results
[ ] P7 — Build clean, ctest 100% pass
[ ] P8 — Commits organized (N logical commits, good messages)
[ ] P9 — github-description.md written
[ ] P9 — explain.md written
[ ] P9 — 02-prs.md updated
[ ] P9 — 04-log.md updated
[ ] P9 — Shadow files current
```
