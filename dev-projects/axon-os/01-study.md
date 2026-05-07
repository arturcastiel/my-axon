# Study — AXON OS: my-axon user-data separation
date: 2026-05-07
project: axon-os
phase: 1-study

---

## Problem Statement

`workspace/` currently serves two distinct roles that need to be separated:

1. **AXON configuration** — programs, preferences, addons, tools, templates, scheduler
   → belongs in `axon.git` (public, versioned with the OS, cloned by all users)

2. **User-generated runtime data** — dev-projects, memory, log, chats, plans, events
   → belongs in a private per-user repo (`my-axon.git`), backed up automatically

Currently both live in `workspace/` as a single nested git repo (`axon-workspace.git`). This means:
- OS config and user private data are backed up to the same repo
- A fresh AXON clone for a new user has no separation point for their own data
- `igap` logs (OS-internal) get mixed with user dev data

---

## Intended Architecture

```
axon-project/
  axon/                      ← axon.git (PUBLIC)
    axon/                    ← kernel (write-gated)
    workspace/               ← AXON config (programs, preferences, tools, addons)
                                tracked directly in axon.git — NO separate git repo
    tools/                   ← OS Python tools
    my-axon/                 ← gitignored by axon.git; user owns this
      dev-projects/          ← code-dev projects (opm-common, etc.)
      memory/
        longterm/
        episodic/
        working/
        local/               ← gitignored in my-axon too (machine-specific)
      log/
        entries/             ← runtime events
        igap/                ← inference gap logs
        turns/               ← turn log
      chats/
      plans/
      events/
      MYAXON.md              ← path map (absolute paths) + user config
```

**`workspace/`** folds back into `axon.git` — it is OS config, not user data.
**`my-axon/`** is the private user repo — backed up to `my-axon.git` on GitHub.

---

## Impact Analysis

### Path keys to split

| Current key | New key | Stays in |
|-------------|---------|----------|
| `W:ws-programs` | `W:ws-programs` (unchanged) | `axon/workspace/programs/` |
| `W:ws-preferences` | `W:ws-preferences` (unchanged) | `axon/workspace/preferences/` |
| `W:ws-addons` | `W:ws-addons` (unchanged) | `axon/workspace/addons/` |
| `W:ws-tools` | `W:ws-tools` (unchanged) | `axon/workspace/tools/` |
| `W:ws-templates` | `W:ws-templates` (unchanged) | `axon/workspace/templates/` |
| `W:ws-compiled` | `W:ws-compiled` (unchanged) | `axon/workspace/programs/compiled/` |
| `W:ws-queue` | `W:ws-queue` (unchanged) | `axon/workspace/scheduler/QUEUE.md` |
| `W:ws-processes` | `W:ws-processes` (unchanged) | `axon/workspace/processes/` |
| `W:ws-dev-projects` | `W:myaxon-dev-projects` | `my-axon/dev-projects/` |
| `W:ws-memory` | `W:myaxon-memory` | `my-axon/memory/` |
| `W:ws-longterm` | `W:myaxon-longterm` | `my-axon/memory/longterm/` |
| `W:ws-episodic` | `W:myaxon-episodic` | `my-axon/memory/episodic/` |
| `W:ws-working` | `W:myaxon-working` | `my-axon/memory/working/` |
| `W:ws-local` | `W:myaxon-local` | `my-axon/memory/local/` |
| `W:ws-log` | `W:myaxon-log` | `my-axon/log/entries/` |
| `W:ws-igap` | `W:myaxon-igap` | `my-axon/log/igap/` |
| `W:ws-chats` | `W:myaxon-chats` | `my-axon/chats/` |
| `W:ws-plans` | `W:myaxon-plans` | `my-axon/plans/` |
| `W:ws-libraries` | `W:myaxon-libraries` | `my-axon/libraries/` |
| `W:ws-generated` | `W:myaxon-generated` | `my-axon/generated/` |
| `W:ws-name` | `W:myaxon-name` | user identity |
| `W:ws-path` | `W:myaxon-path` | absolute path to `my-axon/` |
| `W:ws-os` | unchanged | `axon/axon/` |

### Files to change

| File | Change |
|------|--------|
| `axon/KERNEL-SLIM.md` | Boot step 2: read `MYAXON.md`; add `my-axon-init` detection; rename path keys; workspace layer description |
| `axon/.gitignore` | Add `my-axon/` |
| `workspace/WORKSPACE.md` | Remove user-data `STORE` keys; add note that user data is in `my-axon/` |
| `workspace/programs/workspace-backup.md` | Retarget to `W:myaxon-path` |
| `workspace/programs/save-workspace.md` | Retarget to `my-axon/` |
| `workspace/programs/menu.md` | Update status display, path refs |
| `workspace/programs/status.md` | Update path refs |
| `workspace/programs/session-summary.md` | Update log/igap path refs |
| `workspace/programs/show-memory.md` | Update memory path refs |
| `workspace/programs/code-dev*.md` | Update `W:ws-log`, `W:ws-dev-projects`, `W:ws-igap` refs |
| `workspace/programs/migrate-workspace.md` | Update — now creates `my-axon/` not sibling dir |
| `workspace/programs/quickstart.md` | Update path refs |
| **NEW** `workspace/programs/my-axon-init.md` | Scaffold `my-axon/` from scratch or clone existing repo |

### New `MYAXON.md` (at `my-axon/MYAXON.md`)

Replaces `WORKSPACE.md` path-map role for user data. Contains only `W:myaxon-*` STOREs with absolute paths.

### Boot detection logic

```
# After TOOL(boot): detect my-axon
myaxon-path ← RETRIEVE(L:myaxon-path) | "{axon-root}/my-axon"
IF NOT DIR-EXISTS(myaxon-path) →
  → "⚠  No my-axon/ data folder found."
  answer ← QUERY(user): "Do you have an existing my-axon repo? [yes / no / skip]"
  IF answer ≡ "yes" → EXEC(my-axon-init --mode=clone)
  IF answer ≡ "no"  → EXEC(my-axon-init --mode=fresh)
  IF answer ≡ "skip" → LOG(WARN, "my-axon missing — running in limited mode")
ELSE →
  EXEC(READ("{myaxon-path}/MYAXON.md"))  ← load all W:myaxon-* keys
```

---

## Key Design Decisions

1. **`workspace/` folds into `axon.git`** — no more nested git repo. On a fresh clone, `workspace/` is just part of the OS. Users customise it (add their own programs) but it's still public-safe.

2. **`my-axon/` is gitignored in `axon.git`** — never accidentally included in OS commits.

3. **`igap` program stays in `workspace/programs/`** — it is an OS diagnostic tool. The *logs* it writes go to `my-axon/log/igap/` via `W:myaxon-igap`.

4. **Compiled programs** stay in `workspace/programs/compiled/` — they are derived from source programs in `workspace/`, so they belong with the OS config.

5. **`memory/local/`** stays gitignored inside `my-axon/` — machine-specific keys (`ws-backup-url`, `dev-mode`) must never be pushed.

6. **`my-axon-init`** handles two paths: fresh scaffold (creates all dirs + empty files) and clone (git clone existing repo). Boot calls it automatically when `my-axon/` is absent.
