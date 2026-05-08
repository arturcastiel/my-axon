# Phase 2 — Plan: AXON OS Docs Refresh (v3.0.0)
project: axon-os-docs
date:    2026-05-07

---

## Objective

Update all public-facing docs in the `axon/` repo root to reflect the v3.0.0
architecture: my-axon user-data separation, new setup flow, library-dev suite,
shadow metadata enrichment, and branch-stale detection.

All changes are **doc-only** — no program logic, no Python, no kernel changes.

---

## Dependency Order

```
PR-001 (VERSION) → PR-002 (CHANGELOG) → PR-003 (SETUP) → PR-004 (CONTEXT) → PR-005 (WORKFLOW + README) → PR-006 (EXAMPLE)
                                                                                ↑
                                                              PR-007 (axon.py docstring — optional)
```

PR-001 must land first so the version string is established. CHANGELOG (PR-002)
references VERSION. SETUP (PR-003) is the most invasive rewrite — it must be
consistent before CONTEXT and WORKFLOW reference the setup flow.

---

## PR List

| PR | Title | Files | Depends On | Complexity |
|----|-------|-------|------------|------------|
| PR-001 | Bump VERSION to 3.0.0 | `VERSION` | none | XS |
| PR-002 | Add v3.0.0 CHANGELOG entry | `CHANGELOG.md` | PR-001 | S |
| PR-003 | Rewrite SETUP.md for my-axon flow | `SETUP.md` | PR-002 | M |
| PR-004 | Update CONTEXT.md structure + version | `CONTEXT.md` | PR-003 | M |
| PR-005 | Update WORKFLOW.md + README.md paths | `WORKFLOW.md`, `README.md` | PR-004 | M |
| PR-006 | Fix EXAMPLE.txt directory structure | `EXAMPLE.txt` | PR-005 | S |
| PR-007 | Bump axon.py docstring version | `axon.py` | PR-001 | XS |

---

## Implementation Notes

### PR-001 — VERSION
Single-line file. Change `2.0.0` → `3.0.0`.

### PR-002 — CHANGELOG.md
Prepend a new `## V3.0.0 — 2026-05-07` section above the V2.0.0 entry.

Contents:
- **my-axon user-data separation** — `workspace/` re-absorbed into axon.git as OS config; user data moved to `my-axon/` (gitignored, backed up to private `my-axon.git`)
- **New programs:** `my-axon-init.md` (scaffold/clone my-axon on first boot), `migrate-workspace.md` (one-time migration from old workspace/ layout)
- **Boot change:** AXON auto-detects `my-axon/` absence at boot → dispatches `my-axon-init`; no manual `workspace-backup setup` needed
- **Path key split:** `W:ws-*` (OS config, workspace/) vs `W:myaxon-*` (user data, my-axon/)
- **library-dev suite** (9 programs): `library-dev`, `library-dev-new`, `library-dev-ingest`, `library-dev-explain`, `library-dev-search`, `library-dev-cite`, `library-dev-status`, `library-dev-intersect`, `library-dev-report`
- **Shadow metadata enrichment:** every shadow note now records `Caller-Program:`, `Caller-Project:`, `Branch:`, `Commit:`
- **Branch-stale detection** in `code-dev-study` and `library-dev-explain`
- **igap improvements:** G-01..G-11 — identity lock, mid-loop re-assertion, boot validation, branch-stale, sub-project creation

### PR-003 — SETUP.md (most invasive)

**Two-Repo Design table** — rewrite:

| Repo | What it contains | Visibility |
|---|---|---|
| `axon` | OS core + programs + workspace config | **Public** — clone, pull updates, share |
| `my-axon` *(your repo)* | Your projects, logs, memory, libraries | **Private** — AXON backs this up automatically |

**Key correction:** `workspace/` is no longer excluded from `axon.git` — it IS axon.git now (OS config). User data lives in `my-axon/` which is excluded.

**Step 3** — replace `workspace-backup setup` with:
> AXON detects that `my-axon/` does not exist and automatically runs `my-axon-init`. Answer the prompts:
> - **Existing my-axon repo?** Yes → paste URL, AXON clones it. No → create empty private GitHub repo, paste URL, AXON scaffolds + pushes.

**Remove:** All references to `axon-workspace`, `workspace-backup setup`, and "axon.gitignore excludes workspace/".

### PR-004 — CONTEXT.md

- Version badge: `V2.0.0` → `V3.0.0`, Built date → `2026-05-07`
- **PROJECT STRUCTURE section** — update directory tree:
  - Remove: `workspace/dev-projects/`, `workspace/log/`, `workspace/memory/`
  - Add: `my-axon/` with `dev-projects/`, `memory/`, `log/`, `MYAXON.md`
  - `workspace/` node: label as `← OS config (tracked in axon.git)`
  - Clarify: `workspace/programs/` count and list should include library-dev programs and my-axon-init
- **Add to KEY DESIGN DECISIONS:** my-axon separation rationale (OS config public, user data private, no gitignore dance)

### PR-005 — WORKFLOW.md + README.md

**WORKFLOW.md — Section 3 (Workspace Structure):**
Replace the old block:
```
axon/workspace/
├── programs/
├── dev-projects/
│   └── opm-common/
...
```
With new two-level structure:
```
axon/workspace/           ← OS config (axon.git, public)
│   ├── programs/
│   └── WORKSPACE.md
axon/my-axon/             ← user data (my-axon.git, private)
│   ├── dev-projects/
│   │   └── {slug}/
│   ├── memory/
│   ├── log/
│   └── MYAXON.md
```

**WORKFLOW.md — Section 9 (LGR project phases):**
No path references — check and confirm no `workspace/dev-projects/` hardcoded.

**WORKFLOW.md — Section 10 (Quick Command Reference):**
Confirm library-dev commands are current (they are, just verify).

**README.md:**
- Update workspace structure if present
- Update setup instructions to reference new my-axon flow
- Version badge if any

### PR-006 — EXAMPLE.txt

Replace **DIRECTORY STRUCTURE** section only. Change from:
```
ai-generalworkflow/
├── axon/               ← The OS. Shared. Never copy this per project.
└── workspace/          ← Your project data. One per project.
```
To:
```
axon-project/
├── axon/               ← The OS + programs. Shared. Clone once, use forever.
│   ├── workspace/      ← OS config (tracked in axon.git)
│   └── my-axon/        ← Your private data (gitignored → my-axon.git)
│       ├── dev-projects/
│       ├── memory/
│       ├── log/
│       └── MYAXON.md
```

Replace: "To start a new project: copy workspace/, rename it, update WORKSPACE.md"
With: "To start a new project: run `code-dev new` at the AXON prompt. All projects share one `my-axon/` folder."

The ONE-TIME SETUP section also needs the step change (my-axon-init, not workspace-backup).

The SESSION TRANSCRIPT itself is unchanged — QC workflow logic is still valid.

### PR-007 — axon.py

Update module docstring first line from:
```python
"""AXON CLI — unified entry point for all AXON tools (C: architecture).
```
To include version:
```python
"""AXON CLI v3.0.0 — unified entry point for all AXON tools (C: architecture).
```

---

## Success Criteria

- [ ] `VERSION` reads `3.0.0`
- [ ] `CHANGELOG.md` has a v3.0.0 entry documenting all changes since v2.0.0
- [ ] `SETUP.md` correctly describes the my-axon setup flow; no references to `workspace-backup setup` or `axon-workspace`
- [ ] `CONTEXT.md` shows `my-axon/` in the structure, correct version badge
- [ ] `WORKFLOW.md` Section 3 shows two-level workspace/my-axon layout
- [ ] `EXAMPLE.txt` directory structure matches new layout; "copy workspace/" removed
- [ ] All files are internally consistent with each other
- [ ] No file references `workspace/dev-projects/`, `workspace/log/`, or `workspace/memory/` as user data locations
