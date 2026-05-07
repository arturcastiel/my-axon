# PR-001 — Harmonization Review with revised PR-000

**Created:** 2026-05-06
**Purpose:** Plan to update `lgr-summaryconfig` branch to match the revised PR-000 API (`lgr_name_` / `lgr_name()` instead of `lgr_` / `lgr(lgr_info{})`)
**Source branch:** `lgr-summaryconfig` (tip: `aaa3271e2`)
**Depends on:** PR-000 revised commit (tip: `0d094ea8c` on `lgr-summary`)

---

## What Changed in PR-000 (the breaking change)

| Old PR-000 API | New PR-000 API |
|---|---|
| `optional<EclIO::lgr_info> lgr_{}` private field | `optional<string> lgr_name_{}` private field |
| `.lgr(const lgr_info& info)` setter | `.lgr_name(const string& name)` setter |
| `.lgr()` getter → `optional<lgr_info>&` | `.lgr_name()` getter → `optional<string>&` |
| `operator==` compared `lgr()` (lgr_info) | `operator==` compares `lgr_name()` (string) |
| `operator<` tiebreaker: name → ijk | `operator<` tiebreaker: name only (no ijk) |
| `serializationTestObject`: `lgr_ = lgr_info{"LGR1",{1,2,3}}` | `lgr_name_ = "LGR1"` |
| `SummaryNode.hpp`: `lgr_info` had `operator==` + `serializeOp` | `SummaryNode.hpp`: **identical to master**, no additions |

The key semantic difference: **ijk coordinates are no longer stored in `SummaryConfigNode`**. They are an IO-layer concern (SMSPEC arrays → `ESmry.cpp`). `SummaryConfigNode` stores only the LGR name.

---

## Impact on PR-001 — `SummaryConfig.cpp`

### 1. `keywordLW` (line ~889–896)

**Current code:**
```cpp
const auto lgr = Opm::EclIO::lgr_info { lgr_name, {0, 0, 0} };
// ...
list.push_back(param.namedEntity(wname).lgr(lgr));
```

**Must become:**
```cpp
list.push_back(param.namedEntity(wname).lgr_name(lgr_name));
```

Note: `{0,0,0}` was a sentinel for "whole well, no specific cell" — this concept no longer lives in `SummaryConfigNode`. The `lgr_name` string alone is sufficient. The ijk sentinel is an IO-layer concern if needed (PR-002 or PR-003).

---

### 2. `keywordLB` (line ~1109)

**Current code:**
```cpp
list.push_back(param.lgr(Opm::EclIO::lgr_info { lgr_name, {i, j, k} }));
```

**Must become:**
```cpp
list.push_back(param.lgr_name(lgr_name));
```

Note: `{i, j, k}` were the local cell coordinates parsed from the deck. With PR-000's revised design, these coordinates are **not stored in `SummaryConfigNode`**. The `number_` field already carries the linearized cell index for Block-level vectors. If downstream PRs (PR-002/PR-003) need the raw ijk for a `LB*` node, this is a design question to resolve — it may need to be re-examined (see Open Questions below).

---

### 3. `keywordLC` (line ~1480–1487)

**Current code:**
```cpp
const auto lgr = Opm::EclIO::lgr_info { lgr_name, {i, j, k} };
// ...
list.push_back(param.namedEntity(wname).lgr(lgr));
```

**Must become:**
```cpp
list.push_back(param.namedEntity(wname).lgr_name(lgr_name));
```

Same note as `keywordLB`: ijk for connection-level vectors is dropped from the config layer.

---

### 4. `operator==`, `operator<`, `serializationTestObject` (lines ~2015, 2098–2181)

These are already correct in the revised PR-000 on `lgr-summary`. When PR-001 rebases onto the revised PR-000, these will simply inherit the correct implementation. **No change needed** in PR-001 commits — the PR-000 commit already owns these lines.

However, verify after rebase that no conflict occurs: PR-001's commit `a1fa0c45b` modifies `SummaryConfig.cpp` in overlapping regions. The rebase may need a manual merge.

---

## Impact on PR-001 — `tests/test_SummaryConfigNode.cpp`

### Unit test suite: `LGR` (3 test cases, lines ~185–255)

These are owned by PR-000's revised commit (already updated in `lgr-summary`). After rebase, these tests will be correct. **No change needed in PR-001.**

However, verify there is no duplication: PR-001's commit `8cd484715` also adds to this file. Check for conflicts on rebase.

### Deck-parsing integration tests: `LGR_SummaryConfig_Deck` (lines ~305–420)

These tests call `.lgr()` getter and check `.lgr()->name` and `.lgr()->ijk[0/1/2]`. Every one of these must change:

**Pattern to replace (all occurrences):**
```cpp
hits[0].lgr().has_value()           → hits[0].lgr_name().has_value()
hits[0].lgr()->name == "LGR1"       → *hits[0].lgr_name() == "LGR1"
hits[0].lgr()->ijk[0] == 0          → REMOVE — ijk no longer in config layer
n.lgr().has_value()                 → n.lgr_name().has_value()
n.lgr()->name == "LGR1"             → *n.lgr_name() == "LGR1"
n.lgr()->ijk[0] == 2                → REMOVE — ijk no longer in config layer
```

Affected test cases:
| Test case | Lines | Changes |
|---|---|---|
| `lgr_1lgr_lw_basic` | ~305–315 | Remove ijk checks; `lgr()` → `lgr_name()` |
| `lgr_1lgr_lc_basic` | ~320–350 | Remove ijk checks; `lgr()` → `lgr_name()` |
| `lgr_1lgr_global_well_has_no_lgr` | ~355–367 | `lgr()` → `lgr_name()` |
| `lgr_2lgr_lw_both` | ~380–390 | `lgr()` → `lgr_name()` |
| `lgr_2lgr_lc_both` | ~400–415 | Remove ijk checks; `lgr()` → `lgr_name()` |
| Inline schema tests (`LGR_Schema_Inline`) | ~490–565 | `lgr()` → `lgr_name()`, ijk checks → remove |

---

## Open Questions Before Coding

### Q1 — `LB*` cell coordinates
`keywordLB` currently stores local cell ijk in `lgr_info`. With PR-000's revision, these are dropped from `SummaryConfigNode`. Two options:

- **Option A (preferred):** Store the linearized cell number in `number_` using a 1-based LGR-local index. Downstream (PR-002 SMSPEC, PR-003 evaluator) reads `node.number()` to recover coordinates. This is consistent with how global `BPR` stores a linear cell index.
- **Option B:** Accept that `SummaryConfigNode` loses the ijk and reconstruct them in PR-002/PR-003 from the grid object using `node.lgr_name()` + `node.number()`.

Clarify this before writing PR-001 code.

### Q2 — `LC*` wildcard connections
`keywordLC` uses `{0,0,0}` as a wildcard sentinel for "all connections of this well in this LGR". With ijk gone from `SummaryConfigNode`, how does PR-003/PR-004 distinguish a wildcard `LC*` from an explicit-cell `LC*`? Options:
- Use `number_ == SummaryConfigNode::default_number` as the wildcard signal (already the default).
- Clarify with downstream PR owners before changing `keywordLC`.

---

## Rebase Plan

1. Ensure `lgr-summary` is the revised PR-000 branch (current HEAD: `0d094ea8c`).
2. On `lgr-summaryconfig`, rebase onto the new `lgr-summary` tip:
   ```
   git rebase lgr-summary
   ```
3. Expect conflicts in:
   - `SummaryConfig.cpp`: operator==, operator<, serializationTestObject (resolve by keeping PR-000 version)
   - `test_SummaryConfigNode.cpp`: LGR unit test suite (resolve by keeping PR-000 version)
4. Apply the setter/getter changes listed above to `keywordLW`, `keywordLC`, `keywordLB`.
5. Update all integration tests to use `lgr_name()` instead of `lgr()` / `lgr()->ijk`.
6. Drop all ijk assertions from integration tests (or convert to `number()` assertions if Q1 is resolved as Option A).
7. Build and run `SummaryConfigTests`, `SummaryNode`, `Summary`.

---

## Shadow Files — Stale Entries to Update After Harmonization

| Shadow file | What's stale |
|---|---|
| `SummaryNode.hpp.findings.md` | Still says `lgr_info::operator==` and `serializeOp` were added by PR-000. These were **reverted** — the struct is now identical to master. Update findings log. |
| `SummaryConfig.cpp.findings.md` | Has a single stale entry from 2026-05-04, not yet analysed. Needs full analysis after rebase. |
| `SummaryConfig.hpp.findings.md` | Updated 2026-05-06 — correct, reflects `lgr_name_` design. No action. |
