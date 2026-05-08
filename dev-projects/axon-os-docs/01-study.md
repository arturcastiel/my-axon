# Phase 1 — Study: AXON OS Docs Refresh (v3.0.0)
project: axon-os-docs
date:    2026-05-07
source:  direct audit of axon root files vs current architecture

---

## Goal

Bring all public-facing documentation files in the `axon/` repo root into
alignment with the v3.0.0 architecture (my-axon user-data separation) and
the additions since v2.0.0: library-dev suite, branch-stale detection,
library-dev shadow metadata enrichment, and the new two-repo setup flow.

---

## Current Architecture (What Actually Exists — 2026-05-07)

### Repo layout
```
axon-project/
  axon/                      ← axon.git (PUBLIC)
    axon/                    ← OS kernel (write-gated)
    workspace/               ← AXON OS config — tracked in axon.git
      programs/              ← all programs (code-dev, library-dev, my-axon-init, …)
      WORKSPACE.md           ← W:ws-* keys only (OS paths)
    tools/                   ← Python scripts
    my-axon/                 ← gitignored → my-axon.git (PRIVATE)
      dev-projects/          ← code-dev projects per slug
      memory/{longterm,episodic,working,local}/
      log/{entries,igap,turns}/
      chats/, plans/, libraries/, generated/
      MYAXON.md              ← W:myaxon-* absolute path map
```

### Key architectural facts
- `workspace/` is now **tracked in axon.git** (OS config only — programs, preferences, tools, templates)
- `my-axon/` is **gitignored** from axon.git; backed up to private `my-axon.git`
- Path keys split: `W:ws-*` (OS) vs `W:myaxon-*` (user data)
- Boot detects `my-axon/` absence → auto-dispatches `my-axon-init` (clone or scaffold)
- Programs: `my-axon-init.md`, `migrate-workspace.md` (new since v2.0.0)
- library-dev suite: `library-dev.md`, `library-dev-new.md`, `library-dev-ingest.md`,
  `library-dev-explain.md`, `library-dev-search.md`, `library-dev-cite.md`,
  `library-dev-status.md`, `library-dev-intersect.md`, `library-dev-report.md`
- Shadow metadata enrichment: every shadow note now carries `Caller-Program:`, `Caller-Project:`, `Branch:`, `Commit:`
- Branch-stale detection in: `code-dev-study.md`, `library-dev-explain.md`

---

## Audit: File-by-File Gap Analysis

### VERSION
- **Current:** `2.0.0`
- **Should be:** `3.0.0`
- **Gap:** my-axon separation is a breaking architectural change (user data location, setup flow, path keys). Warrants major version bump.

### CHANGELOG.md
- **Current:** Two entries — V2.0.0 (2026-05-06) and V0.1.0 (2026-05-02).
- **Missing:** V3.0.0 entry covering:
  - my-axon user-data separation (new private repo)
  - workspace/ re-absorbed into axon.git (OS config)
  - new programs: `my-axon-init`, `migrate-workspace`
  - boot auto-detects my-axon/ absence and dispatches my-axon-init
  - path key split: W:ws-* vs W:myaxon-*
  - library-dev suite (9 programs) — though mentioned in WORKFLOW, not in changelog
  - shadow metadata enrichment (Caller-Program, Caller-Project, Branch, Commit)
  - branch-stale detection in code-dev-study and library-dev-explain
  - igap improvements (G-01..G-11, IMP-001..006)

### CONTEXT.md
- **Current:** Built 2026-05-02, Version V2.0.0. Shows `workspace/dev-projects/`, `workspace/log/`, `workspace/memory/` as user data paths.
- **Gaps:**
  - Project structure shows old layout (`workspace/` = user data, not OS config)
  - No mention of `my-axon/` or MYAXON.md
  - Two-repo design implicit in structure but not named (axon + axon-workspace was the old model)
  - Dev-projects now at `my-axon/dev-projects/`; logs at `my-axon/log/`
  - Missing: library-dev, shadow metadata, branch-stale, my-axon-init
  - Version badge should be 3.0.0

### SETUP.md
- **Current:** Documents old two-repo design: `axon` (public) + `axon-workspace` (private).
  - Says: "The axon repo .gitignore already excludes workspace/"
  - Setup step 3: run `workspace-backup setup`
- **Gaps — this is the most outdated file:**
  - Old model: workspace/ gitignored → user runs workspace-backup setup → links axon-workspace.git
  - New model: workspace/ tracked in axon.git; my-axon/ gitignored → boot auto-runs my-axon-init → links my-axon.git
  - Two-repo table: `axon-workspace` → `my-axon`
  - "workspace/ is excluded from axon.git" → WRONG. workspace/ IS now in axon.git.
  - Step 3 ("workspace-backup setup") → replaced by: "AXON auto-detects no my-axon/ and runs my-axon-init"
  - The "Every Session" section is probably fine but needs path updates

### README.md
- **Current:** Standard project overview. References library-dev (was added in last WORKFLOW commit). Two-repo model may be referenced.
- **Gaps (likely):** Setup instructions point to old `workspace-backup setup` flow; workspace structure description outdated. Version badge.

### WORKFLOW.md
- **Current:** 457 lines. Section 3 "Workspace Structure" shows `axon/workspace/` as user space with dev-projects, log, memory all under workspace/. Section 9 (LGR project) references workspace paths.
- **Gaps:**
  - Section 3 structure diagram: all `workspace/dev-projects/`, `workspace/log/` → now `my-axon/dev-projects/`, `my-axon/log/`
  - No mention of my-axon or the two-repo design
  - library-dev table in section 10 is accurate but may be missing new commands (shadow metadata, branch-stale not user-facing)
  - Workspace path in SETUP step references may be wrong

### EXAMPLE.txt
- **Current:** Shows old directory structure with `workspace/` as user project folder (copy it per project). This was the pre-v0.1.0 model where workspace was per-project, not shared.
- **Gaps:**
  - The directory structure shown is from before even v0.1.0 — extremely stale
  - Should reflect: `axon/` (shared OS) + `my-axon/` (private user data, one for all projects)
  - The "To start a new project: copy workspace/, rename it" instruction is completely wrong
  - The QC example itself (session transcript) is still valid — the workflow logic hasn't changed

### axon.py
- **Current:** Functional CLI tool. Docstring version not bumped. No structural issues.
- **Gaps:** Minor — version string in module docstring should reflect 3.0.0 or be removed. No functional changes needed.

---

## Constraints

- `axon/` (kernel, compiler, core) — no changes needed; architecture unchanged
- `tools/` Python scripts — no changes needed
- `workspace/programs/` — no changes needed (already updated throughout axon-os project)
- EXAMPLE.txt session transcript — the QC example workflow is still accurate; only the directory structure header needs updating
- All changes are doc-only: no program logic, no Python, no kernel changes

---

## Confidence: HIGH

Every gap is concrete and bounded. The new architecture is fully implemented — the docs just haven't caught up. No design decisions remain open.
