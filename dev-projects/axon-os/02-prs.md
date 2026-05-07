# PR List — AXON OS: my-axon separation
date: 2026-05-07
project: axon-os

---

## PR-000 — `my-axon-init` program
**Branch:** `my-axon-init`
**Status:** not-started
**Depends on:** nothing
**Files:**
- NEW `workspace/programs/my-axon-init.md`

**Scope:** New program that scaffolds a fresh `my-axon/` directory tree OR clones an existing repo.
Modes: `--mode=fresh` (create dirs + empty files) and `--mode=clone` (git clone URL).
Creates: `MYAXON.md`, `dev-projects/`, `memory/{longterm,episodic,working,local}/`, `log/{entries,igap,turns}/`, `chats/`, `plans/`, `events/`, `.gitignore`.
Sets all `W:myaxon-*` keys in working memory after scaffold.

---

## PR-001 — Kernel: `my-axon` boot detection + `W:myaxon-*` keys
**Branch:** `my-axon-kernel`
**Status:** not-started
**Depends on:** PR-000 (my-axon-init must exist for boot to dispatch to it)
**Files:**
- `axon/KERNEL-SLIM.md`
- `axon/.gitignore`

**Scope:**
- Add `my-axon/` to `axon/.gitignore`
- Boot step 2: after `TOOL(boot)`, detect `my-axon/`; if absent → ask user and dispatch `my-axon-init`; if present → read `MYAXON.md` and `STORE(W:myaxon-*)` keys
- Add `W:myaxon-*` key definitions to PATH MAP / memory scopes section
- Update Layer 2/3 architecture description: `workspace/` = OS config, `my-axon/` = user data
- G-10 workspace path validation: also validate `W:myaxon-path` if set

---

## PR-002 — Config layer: `WORKSPACE.md` trim + `MYAXON.md` template
**Branch:** `my-axon-config`
**Status:** not-started
**Depends on:** PR-001
**Files:**
- `workspace/WORKSPACE.md`
- `MYAXON.md` template (written by `my-axon-init`, not a static file)

**Scope:**
- Remove all user-data `STORE` keys from `WORKSPACE.md` (`W:ws-log`, `W:ws-dev-projects`, `W:ws-memory`, `W:ws-chats`, `W:ws-plans`, `W:ws-episodic`, `W:ws-longterm`, `W:ws-working`, `W:ws-local`, `W:ws-igap`, `W:ws-libraries`, `W:ws-generated`, `W:ws-name`, `W:ws-path`)
- Keep only: `W:ws-programs`, `W:ws-preferences`, `W:ws-addons`, `W:ws-tools`, `W:ws-templates`, `W:ws-compiled`, `W:ws-queue`, `W:ws-processes`, `W:ws-os`
- Add note: "User runtime data → see `my-axon/MYAXON.md`"
- `MYAXON.md` template (used by `my-axon-init`): contains all `W:myaxon-*` STOREs with absolute paths, user name, backup remote

---

## PR-003 — Programs: rename user-data path refs
**Branch:** `my-axon-programs`
**Status:** not-started
**Depends on:** PR-001 (W:myaxon-* keys must be defined in kernel first)
**Files:**
- `workspace/programs/code-dev.md`
- `workspace/programs/code-dev-study.md`
- `workspace/programs/code-dev-plan.md`
- `workspace/programs/code-dev-pr.md`
- `workspace/programs/code-dev-log.md`
- `workspace/programs/code-dev-shadow.md`
- `workspace/programs/code-dev-audit.md`
- `workspace/programs/code-dev-explain.md`
- `workspace/programs/code-dev-init.md`
- `workspace/programs/session-summary.md`
- `workspace/programs/show-memory.md`
- `workspace/programs/menu.md`
- `workspace/programs/status.md`
- `workspace/programs/quickstart.md`
- `workspace/programs/migrate-workspace.md`

**Scope:** Mechanical rename — every `W:ws-log` → `W:myaxon-log`, `W:ws-dev-projects` → `W:myaxon-dev-projects`, `W:ws-igap` → `W:myaxon-igap`, etc. per the mapping table in 01-study.md.
Also update any hardcoded `"workspace/dev-projects/"`, `"workspace/log/"`, `"workspace/memory/"` paths to use `W:myaxon-*` keys.

---

## PR-004 — Backup: retarget to `my-axon/`
**Branch:** `my-axon-backup`
**Status:** not-started
**Depends on:** PR-003
**Files:**
- `workspace/programs/workspace-backup.md`
- `workspace/programs/save-workspace.md`

**Scope:**
- `workspace-backup`: change target from `W:ws-path` (workspace/) to `W:myaxon-path` (my-axon/)
- `save-workspace`: same retarget
- Rename program to `my-axon-backup` (or keep `workspace-backup` as alias for compatibility)
- `memory/local/` keys: `ws-backup-url.md` → `myaxon-backup-url.md`, `ws-backup-enabled.md` → `myaxon-backup-enabled.md`
- Auto-push on boot: retarget to `my-axon/` git repo

---

## PR-005 — Migration: move data + dissolve nested workspace git
**Branch:** `my-axon-migrate`
**Status:** not-started
**Depends on:** PR-000..PR-004
**Files:**
- `workspace/programs/migrate-workspace.md` (rewrite for new purpose)

**Scope:**
- Rewrite `migrate-workspace.md`: new purpose is to move existing data from `workspace/{dev-projects,memory,log,chats,plans,events}` into `my-axon/`
- Program outputs shell commands (human runs):
  1. `mkdir -p my-axon/` (scaffold via my-axon-init fresh)
  2. `mv workspace/dev-projects my-axon/`
  3. `mv workspace/memory my-axon/`
  4. `mv workspace/log my-axon/`
  5. `mv workspace/chats my-axon/`
  6. `mv workspace/plans my-axon/`
  7. `mv workspace/events my-axon/`
  8. Remove `workspace/.git` (workspace folds back into axon.git): `rm -rf workspace/.git`
  9. `cd axon && git add workspace/` (re-track in axon.git)
  10. Init `my-axon` as its own git repo + set remote + push
- After migration: `workspace/` is clean OS config only, tracked in axon.git
