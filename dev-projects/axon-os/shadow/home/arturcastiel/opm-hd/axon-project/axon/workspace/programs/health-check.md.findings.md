---
name: health-check.md
description: Data-driven tool health sweep over REGISTRY.json with auto-remediation
type: findings
hash: 5996ff0566fd
git-branch: main
updated: 2026-05-08
caller-program: code-dev-study
caller-project: axon-os
---

## Summary

Data-driven tool health sweep over REGISTRY.json with auto-remediation. Lives at workspace/programs/health-check.md.

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
