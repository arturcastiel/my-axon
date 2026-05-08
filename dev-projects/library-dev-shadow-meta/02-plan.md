# Phase 2 — Plan: Library-Dev Shadow Metadata
project: library-dev-shadow-meta
date:    2026-05-07

---

## Objective

Add AXON-standard provenance metadata to shadow notes produced by `library-dev-ingest` so that each shadow note is traceable to its caller program and project.

---

## Architecture

**Single file changed:** `workspace/programs/library-dev-ingest.md`
**Change location:** WRITE block, lines 169–182 (the shadow note header)

The fix is purely additive — two new fields inserted into the existing `> Key: Value` header block. No logic changes, no new variables needed (both `lib-name` and the program name are already in scope).

**No dependency on code changes in other programs.** `library-dev-explain.md`, `library-dev-search.md`, and `library-dev-cite.md` read shadow notes but do not inspect the header fields — they are not broken and do not need updates in PR-001.

---

## PR List

| PR | Title | Scope | Depends On | Complexity |
|----|-------|-------|------------|------------|
| PR-001 | Add caller/project/branch metadata to shadow header | `library-dev-ingest.md` WRITE block + git probe | none | S |
| PR-002 | Add branch-stale detection to library-dev-explain | `library-dev-explain.md` | PR-001 | M |

---

## Implementation Notes

### PR-001 — IMPLEMENTED

Added to the WRITE block in `library-dev-ingest.md`:

```
> Caller-Program: library-dev-ingest
> Caller-Project: {lib-name}
> Branch:  {ingest-branch}
> Commit:  {ingest-commit}
```

Branch/commit logic (added before WRITE):
```
assoc-repo ← meta.associated-codebase | ∅
IF assoc-repo ≠ ∅ →
  ingest-branch ← TOOL(shell, "git -C '{assoc-repo}' branch --show-current") | ∅
  ingest-commit ← TOOL(shell, "git -C '{assoc-repo}' rev-parse --short HEAD") | ∅
  IF ingest-branch ≡ ∅ OR ingest-branch.strip ≡ "" → ingest-branch ← "none"
  IF ingest-commit ≡ ∅ OR ingest-commit.strip ≡ "" → ingest-commit ← "none"
ELSE →
  ingest-branch ← "none"
  ingest-commit ← "none"
```

- When `associated-codebase` is not set in `_meta.md` → both fields write `none`
- When git fails (detached HEAD, not a repo, empty output) → falls back to `none`
- No crash path — fully defensive

### PR-002 (deferred — optional)

With `Branch:` / `Commit:` now written to every shadow note (PR-001), PR-002 adds the consumer side:
- `library-dev-explain` reads `Branch:` from the shadow header
- Compares vs current `git branch --show-current` of `associated-codebase`
- If different → surface `⚠ branch-stale: shadow built on {shadow-branch}, current: {current-branch}`

PR-002 is **optional** — tracked here but not required before closing this project.

---

## Success Criteria

- [x] Every shadow note written by `library-dev-ingest` includes `Caller-Program:` and `Caller-Project:` fields
- [x] `Branch:` and `Commit:` fields written — `none` when no associated-codebase or git fails
- [x] Fields follow `> Key: Value` convention consistent with existing header
- [ ] No regression in ingest pipeline (INDEX update, meta counts, article loop continue to work)
- [ ] PR-001 reviewed and merged to axon.git
