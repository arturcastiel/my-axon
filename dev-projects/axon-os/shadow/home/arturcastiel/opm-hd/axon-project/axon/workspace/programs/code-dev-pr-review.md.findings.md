---
name: code-dev-pr-review.md
description: 9-phase PR review workflow: context load → study → conflict analysis → harmonize → rebase → execute → verify → commit → document
type: findings
hash: 6078bbd114ac
git-branch: main
updated: 2026-05-08
caller-program: code-dev-study
caller-project: axon-os
---

## Summary

9-phase repeatable workflow for taking a branch from "needs review" or "diverged" to "clean, rebased, tested, documented, and ready to push". Distilled from opm-common PR-000 / PR-001 harmonization work. Shadow enforced at every file read. Build/test/push are human-only.

## Key Structures

| Name | Kind | Purpose |
|------|------|---------|
| PHASE 1 | section | Context load: PR tracking, log, upstream state, file list |
| PHASE 2 | section | Study: shadow every touched + dependency file |
| PHASE 3 | section | Conflict analysis: API drift, superseded work, design decisions |
| PHASE 4 | section | Harmonization plan: numbered steps with file/region/change/verify |
| PHASE 5 | section | Rebase: human-only, conflict map applied |
| PHASE 6 | section | Execution: apply plan steps, update shadow per file |
| PHASE 7 | section | Verification: grep sweeps + build + ctest (human-only) |
| PHASE 8 | section | Commit organization: git reset --mixed → logical groups |
| PHASE 9 | section | Documentation: GitHub description + explain + tracking update |

## Findings Log

| Date | Context | Finding |
|------|---------|---------|
| 2026-05-08 | code-dev-study / axon-os | Program created from code-dev-pr-review plan (my-axon/plans/code-dev-pr-review.md) |
| 2026-05-08 | code-dev-study / axon-os | Help screen added: PURPOSE + 9-phase WORKFLOW + CONSTRAINTS + EXAMPLE shown at top |
| 2026-05-08 | code-dev-study / axon-os | Wired into code-dev.md: dispatch via 'code-dev review [N]', listed in subprograms and command list |
| 2026-05-08 | code-dev-study / axon-os | Shadow gate enforced at P2 — every touched file checked before reading source |
| 2026-05-08 | code-dev-study / axon-os | Phase resume: W:code-dev-pr-review-phase allows starting at any phase (1-9) |
