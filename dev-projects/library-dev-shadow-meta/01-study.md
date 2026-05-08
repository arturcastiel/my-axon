# Phase 1 — Study: Library-Dev Shadow Metadata
project: library-dev-shadow-meta
date:    2026-05-07
source:  igap G-LIB-001, direct program audit

---

## Goal

Add provenance metadata to shadow notes written by `library-dev-ingest` so that:
1. Each shadow note records **which AXON program** wrote it
2. Each shadow note records **which library project** it belongs to
3. (Optional, deferred) Record associated codebase git branch/commit if library is tied to a repo

---

## Key Concepts

- Shadow note provenance
- `--caller-program` / `--caller-project` metadata fields
- `library-dev-ingest` WRITE block
- Branch-stale detection (deferred)

---

## Tech Stack

Language: AXON pseudocode
Target file: `workspace/programs/library-dev-ingest.md`
Reference implementations: `code-dev-shadow.md`, `code-dev-study.md`

---

## Findings

### What `library-dev-ingest` currently writes

The shadow note header block (lines 169–182):
```
# Shadow: {title-final}
> File:    {filename}
> Authors: {authors-final}
> DOI:     {doi-final}
> Year:    {year-auto | 'unknown'}
> Library: {lib-name}
> Shadowed: {ts.iso}
```

**Missing metadata:**
- No `caller-program` field
- No `caller-project` (library name is present as `Library:` but not in the AXON metadata key convention)
- No git branch / commit (library articles are not source files — git metadata is less critical here)

### How code-dev passes metadata (reference)

`code-dev-shadow.md` and `code-dev-study.md` call:
```
TOOL(shadow, init,
  "--file {abs-path}
   --branch {git-branch} --commit {git-commit} --commit-msg {git-msg[:80]}
   --caller-program code-dev-shadow --caller-project {RETRIEVE(W:code-dev-project)}")
```

Library-dev does **not** call `TOOL(shadow, init)` at all — it constructs shadow notes with `DERIVE()` + `WRITE()` directly.

### Why the gap exists

`library-dev-ingest` predates the `--caller-program/project` convention added during G-LIB-001 parent work. It was written as a standalone article-ingestion pipeline, not as part of the shadow tool ecosystem.

### Impact

- No caller tracing: cannot tell which program wrote a shadow note from the note alone
- Cannot filter/search shadows by project at the AXON OS level
- Branch-stale detection (added in code-dev-study for code shadows) has no equivalent for library shadows

---

## Constraints

- Fix must not break `library-dev-ingest` ingestion pipeline (article loop, INDEX update, meta count)
- Git metadata is optional — libraries are not tied to a specific codebase branch
- The `Library:` field already records the library name — the fix should add AXON-standard keys alongside it, not replace it

---

## Confidence: HIGH

The fix is fully contained to `library-dev-ingest.md`, specifically the WRITE block at lines 169–182. No other files are affected for Option C. Option A (full `TOOL(shadow, init)` refactor) would also touch `library-dev-explain.md` if shadow-hit detection is added.
