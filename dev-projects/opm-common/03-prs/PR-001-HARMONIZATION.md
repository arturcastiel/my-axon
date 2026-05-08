# PR-001 — Harmonization Plan: rebase onto revised PR-000

**Created:** 2026-05-07
**Status:** plan — ready to execute
**Branch to update:** `lgr-summaryconfig` (tip: `aaa3271e2`)
**Rebase target:** `lgr-summary` (tip: `0066645cd`, revised PR-000)

---

## Root Cause of Divergence

`lgr-summaryconfig` was built on the OLD PR-000 (`4ac7f0a5c`) which used a different API:

| Dimension | Old PR-000 (what PR-001 was built on) | Revised PR-000 (current `lgr-summary`) |
|---|---|---|
| Config field | `optional<EclIO::lgr_info> lgr_{}` | `optional<string> lgr_name_{}` |
| Setter | `.lgr(const lgr_info&)` | `.lgr_name(const string&)` |
| Getter | `.lgr()` → `optional<lgr_info>&` | `.lgr_name()` → `optional<string>&` |
| ijk in config | Yes — stored in `lgr_info.ijk` | No — IO-layer concern only |
| `SummaryNode.hpp` | `lgr_info` had `operator==` + `serializeOp` (added by old PR-000) | **Unchanged from master** |
| `operator<` | entity-first, then LGR | LGR-first (via `make_tuple`) |

Every use of `.lgr()` / `.lgr(lgr_info{})` in PR-001 must become `.lgr_name()` / `.lgr_name(string)`.

---

## What Must Change in PR-001

### Step 1 — Rebase

```bash
git checkout lgr-summaryconfig
git rebase lgr-summary
```

Expect conflicts in 3 files. Resolve as described below.

---

### Step 2 — Resolve conflict: `opm/io/eclipse/SummaryNode.hpp`

**Conflict:** PR-001 added `lgr_info::operator==` and `lgr_info::serializeOp`.
Revised PR-000 leaves `SummaryNode.hpp` **identical to master**.

**Resolution:** Keep PR-000's version — drop both additions entirely.
Neither is needed: `operator==` compared `lgr_info` structs (gone), `serializeOp` serialized `lgr_` field (gone).

---

### Step 3 — Resolve conflict: `SummaryConfig.cpp` — comparator and serialization regions

Three regions will conflict:

**`serializationTestObject()`:**
- PR-001 has: `result.lgr_ = EclIO::lgr_info{ "LGR1", {1, 2, 3} };`
- PR-000 has: `result.lgr_name_ = std::string{ "LGR1" };`
- **Keep PR-000's version.**

**`operator==(const SummaryConfigNode&, ...)`:**
- PR-001 has: `&& (lhs.lgr() == rhs.lgr());` in all 3 category groups
- PR-000 has: `&& (lhs.lgr_name() == rhs.lgr_name());`
- **Keep PR-000's version.**

**`operator<(const SummaryConfigNode&, ...)`:**
- PR-001 has: verbose multi-if chain using `lhs.lgr()` / `rl.has_value()`
- PR-000 has: `make_tuple(lgr_name, ...)` — LGR-first, one line per case
- **Keep PR-000's version.**

---

### Step 4 — Fix `keywordLW` (~line 864 in PR-001 branch)

**Current:**
```cpp
const auto lgr = Opm::EclIO::lgr_info { lgr_name, {0, 0, 0} };
// ...
list.push_back(param.namedEntity(wname).lgr(lgr));
```

**New:**
```cpp
list.push_back(param.namedEntity(wname).lgr_name(lgr_name));
```

Drop the `lgr` local variable entirely. The `{0,0,0}` sentinel was an artefact of the old design — well-level LGR vectors have no meaningful cell coordinates in the config layer.

---

### Step 5 — Fix `keywordLB` (~line 1094 in PR-001 branch)

**Current:**
```cpp
const auto i = record.getItem(1).get<int>(0);
const auto j = record.getItem(2).get<int>(0);
const auto k = record.getItem(3).get<int>(0);
list.push_back(param.lgr(Opm::EclIO::lgr_info { lgr_name, {i, j, k} }));
```

**New:**
```cpp
list.push_back(param.lgr_name(lgr_name));
```

Drop `i`, `j`, `k` and the `lgr_info` construction entirely.

**Design note:** ijk for LB* vectors cannot be linearized at parse time (no LGR GridDims available).
The config layer stores only the LGR name for routing. Cell identification for evaluation is a
PR-006 / IO-layer concern resolved from SMSPEC arrays or the LGR grid object.

---

### Step 6 — Fix `keywordLC` (~line 1450 in PR-001 branch)

**Current:**
```cpp
const auto lgr = Opm::EclIO::lgr_info { lgr_name, {i, j, k} };
// ...
list.push_back(param.namedEntity(wname).lgr(lgr));
```

**New:**
```cpp
list.push_back(param.namedEntity(wname).lgr_name(lgr_name));
```

Keep the `coords_defaulted` / `i`, `j`, `k` reads only if they are needed for future validation.
If not used, drop them. The `lgr_info` local variable is gone regardless.

---

### Step 7 — Resolve conflict: `test_SummaryConfigNode.cpp` — LGR suite

**Conflict:** PR-001 added `LGR` suite with 3 test cases (`lgr_field_roundtrip`,
`lgr_dedup_distinct_lgrs`, `lgr_no_lgr_sorts_before_lgr`). Revised PR-000 now owns this suite
with 6 test cases (original 3 + 3 review-response additions).

**Resolution:** Keep PR-000's version of the entire `LGR` suite (6 tests).
Drop PR-001's 3 LGR tests — they are a subset of what PR-000 now provides.

---

### Step 8 — Update `LGR_SummaryConfig_Deck` integration tests

All assertions using `.lgr()` getter or `->ijk` must change.

**Pattern replacements (apply to all 6 test cases):**

| Old | New |
|---|---|
| `hits[0].lgr().has_value()` | `hits[0].lgr_name().has_value()` |
| `hits[0].lgr()->name == "LGR1"` | `*hits[0].lgr_name() == "LGR1"` |
| `hits[0].lgr()->ijk[0] == 0` | **Remove** — ijk not in config layer |
| `n.lgr().has_value()` | `n.lgr_name().has_value()` |
| `n.lgr()->name == "LGR1"` | `*n.lgr_name() == "LGR1"` |
| `n.lgr()->ijk[N] == X` | **Remove** |
| `!it->lgr().has_value()` | `!it->lgr_name().has_value()` |

Affected test cases:
- `lgr_1lgr_lw_node_populated` — remove 3 ijk assertions, update getter
- `lgr_1lgr_lc_node_populated` — remove 3 ijk assertions, update getter
- `lgr_1lgr_lb_node_populated` — remove 3 ijk assertions, update getter
- `lgr_1lgr_global_well_has_no_lgr` — update getter only
- `lgr_2lgr_lw_two_nodes_not_deduped` — update getter
- `lgr_2lgr_lb_dedup_by_lgr_name` — update getter, remove ijk assertions

---

### Step 9 — Update `LGR_Schema_Inline` inline tests

Same pattern as Step 8. All `.lgr()` → `.lgr_name()`, all `->ijk` assertions removed.

Affected test cases:
- `lw_single_record_produces_node`
- `lw_multi_record_produces_two_nodes`
- `lc_multi_record_produces_nodes`
- `lb_multi_record_produces_nodes`
- `lw_regex_matches_lwstat`

---

### Step 10 — Build and test

```bash
cd build
ctest -R "^(SummaryNode|SummaryConfigTests|Serialization)$" --output-on-failure
```

All 3 must pass. Confirm test count in SummaryConfigTests has grown (PR-000's 6 LGR tests +
PR-001's ParseKeywords/Category, NoSumLgr, LGR_SummaryConfig_Deck, LGR_Schema_Inline suites).

---

## Conflict Map Summary

| File | Conflict type | Resolution |
|---|---|---|
| `SummaryNode.hpp` | PR-001 added operator== + serializeOp to lgr_info | Keep master (PR-000 reverted these) |
| `SummaryConfig.cpp` — serializationTestObject | `lgr_` vs `lgr_name_` | Keep PR-000 |
| `SummaryConfig.cpp` — operator== | `.lgr()` vs `.lgr_name()` | Keep PR-000 |
| `SummaryConfig.cpp` — operator< | verbose vs make_tuple | Keep PR-000 |
| `SummaryConfig.cpp` — keywordLW/LC/LB | `.lgr(lgr_info)` → `.lgr_name(string)` | Rewrite (Step 4–6) |
| `test_SummaryConfigNode.cpp` — LGR suite | PR-001 has 3, PR-000 has 6 | Keep PR-000's 6 |
| `test_SummaryConfigNode.cpp` — Deck/Schema tests | `.lgr()` API + ijk assertions | Update (Step 8–9) |

---

## PR-001 Goals: Compliance Check After Harmonization

| Goal | Impact | Status after plan |
|---|---|---|
| Parse `LW*/LC*/LB*` keywords via `keywordLW/LC/LB` | API changes only, logic unchanged | ✓ preserved |
| Filter wells by `get_lgr_well_tag()` | Logic unchanged | ✓ preserved |
| `parseKeywordCategory` case 'L': | No conflict (PR-001 owns this) | ✓ preserved |
| `handleKW` dispatch | No conflict | ✓ preserved |
| `NOSUMLGR` field + processing | No conflict with PR-000 | ✓ preserved |
| Keyword JSON schemas (4 files) | No conflict | ✓ preserved |
| 28 unit + integration tests | LGR suite dropped (now in PR-000), rest updated | ✓ preserved |
| `lgr_name_` set on every LW*/LC*/LB* node | Setter call updated to new API | ✓ preserved |
