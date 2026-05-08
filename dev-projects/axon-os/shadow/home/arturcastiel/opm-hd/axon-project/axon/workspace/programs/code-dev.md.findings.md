---
name: code-dev.md
description: 5-phase code development harness: study → plan → PR specs → log → audit
type: findings
hash: 2eac612360e3
git-branch: main
updated: 2026-05-08
caller-program: code-dev-study
caller-project: axon-os
---

## Summary

5-phase code development harness: study → plan → PR specs → log → audit. Lives at workspace/programs/code-dev.md.

## Key Structures

| Name | Kind | Purpose |
|------|------|---------|
| ## HELP | section | Usage, inputs, outputs, tips — shown in program preamble |
| ## IDENTITY LOCK | section | ASSERT cognition-frame before any execution |
| ## GUARD | section | Pre-conditions — FAIL fast if project not loaded |
| ## LOAD CONTEXT | section | Read project meta, shadow stats, git state |
| ## OUTPUT | section | Main render block — help screen + live data |

## Findings Log

| Date | Context | Finding |
|------|---------|---------|
| 2026-05-08 | code-dev-study / axon-os | Help screen added: PURPOSE + WORKFLOW + SUBPROGRAMS + EXAMPLE sections shown at top of every run |
| 2026-05-08 | code-dev-study / axon-os | Shadow gate enforced in code-dev: fires before study/plan/pr/log/audit when codebase is set |
