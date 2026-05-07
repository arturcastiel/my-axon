# Plan — AXON OS: my-axon user-data separation
date: 2026-05-07
project: axon-os
based-on: 01-study.md

---

## Goal

Split `workspace/` into two clean layers:
- **`workspace/`** — AXON OS config (programs, preferences, addons) → tracked in `axon.git`
- **`my-axon/`** — private user runtime data (dev-projects, memory, log, chats) → tracked in `my-axon.git`

---

## Approach

Incremental PRs ordered by dependency. No PR breaks the system on its own — each leaves AXON in a working state.

**Order:**
1. New program `my-axon-init` — can ship independently, no kernel change yet
2. Kernel + `.gitignore` changes — add `my-axon/` detection to boot, rename `W:myaxon-*` keys
3. Update `WORKSPACE.md` + create `MYAXON.md` template — config layer clean
4. Update all programs that reference user-data path keys
5. Migrate existing `workspace/` user data to `my-axon/` on disk + update `workspace/.git` situation
6. Update `workspace-backup` / `save-workspace` to target `my-axon/` only

---

## PR List (see 02-prs.md)

| PR | Title | Files |
|----|-------|-------|
| PR-000 | `my-axon-init` program | NEW `programs/my-axon-init.md` |
| PR-001 | Kernel: `my-axon` boot detection + `W:myaxon-*` keys | `axon/KERNEL-SLIM.md`, `axon/.gitignore` |
| PR-002 | Config layer: `WORKSPACE.md` trim + `MYAXON.md` template | `workspace/WORKSPACE.md`, NEW `my-axon/MYAXON.md` template in init |
| PR-003 | Programs: rename user-data path refs to `W:myaxon-*` | all `code-dev*.md`, `session-summary.md`, `show-memory.md`, `menu.md`, `status.md`, `quickstart.md` |
| PR-004 | Backup: retarget `workspace-backup` + `save-workspace` to `my-axon/` | `workspace-backup.md`, `save-workspace.md` |
| PR-005 | Migration: move existing user data from `workspace/` to `my-axon/`; dissolve nested git | shell commands (human executes) + `migrate-workspace.md` update |
