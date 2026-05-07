# Study — opm-common LGR Summary Vectors
phase:    1-study
updated:  2026-05-06
axon-confidence: 10/10
user-satisfaction: 10/10
status:   complete — all studies done, moved to phase 4-log

---

## Goal

Add LGR (Local Grid Refinement) summary vector support to OPM's UNSMRY output.
Target: LW* (local well), LC* (local connection), LB* (local block) vectors
written to the global UNSMRY file at global report steps, plus correct SMSPEC
metadata arrays (LGRS, NUMLX, NUMLY, NUMLZ, LGRNAMES).

**Permanently out of scope:** local timestepping, ROOT.LGR file, LGRTIMES, LGRVEC,
ACTIONX with LGR vectors, nested LGR, Eclipse 300 specifics.

---

## Tech Stack

- Language: C++17
- Repo: opm-common (`/home/arturcastiel/opm-hd/projects/opm-development/opm-common`)
- Also affected: opm-simulators (PR6 only)
- Test framework: Boost.Test
- Build: cmake
- Keyword schemas: JSON files in `opm/input/eclipse/share/keywords/`

---

## Key Concepts

- **LGR (Local Grid Refinement):** sub-grid within a cell block of the global grid. Wells can be inside an LGR.
- **LW*/LC*/LB*:** Eclipse summary keyword families for LGR well/connection/block vectors
- **SMSPEC:** binary file listing all summary vector metadata (keyword, well name, cell index, units)
- **UNSMRY:** binary file containing time series values for each vector
- **SummaryConfigNode:** config-layer object representing one requested summary vector
- **SummaryNode (EclIO):** IO-layer object used when writing SMSPEC/UNSMRY
- **lgr_info:** struct `{ std::string name; std::array<int,3> ijk; }` — LGR identity attached to SummaryConfigNode/SummaryNode

---

## Pipeline (confirmed)

```
SUMMARY deck section
      │
      ▼
Parser (validates against JSON schema files in share/keywords/)
      │  ← LOCAL_WELL_PROBE / LOCAL_CONNECTION_PROBE / LOCAL_BLOCK_PROBE required
      ▼
SummaryConfig constructor → handleKW() for each keyword
      │  parseKeywordCategory() routes L* keywords
      │  handleKW() dispatches to keywordLW / keywordLC / keywordLB
      ▼
SummaryConfig::m_keywords (vector<SummaryConfigNode>)
      │  each node carries lgr_ field with {name, ijk}
      ▼
OutputStream::SummarySpecification::write()   [PR2]
      │  writes SMSPEC arrays: LGRS, NUMLX/Y/Z, LGRNAMES
      ▼
Summary::Factory::create()   [PR3/PR4/PR6]
      │  isLgrWellValue / isLgrConnectionValue / isLgrBlockValue
      ▼
UNSMRY time series values written per report step
```

---

## Confirmed Codebase Facts (do not re-grep)

| Item | File | Lines | Status |
|---|---|---|---|
| `lgr_info` struct: `name` + `ijk` (`std::array<int,3>`) | `opm/io/eclipse/SummaryNode.hpp` | 31-34 | ✅ |
| `std::optional<lgr_info> lgr {}` field in `SummaryNode` | `opm/io/eclipse/SummaryNode.hpp` | 68 | ✅ |
| `lgr_info::operator==` (= default) | `opm/io/eclipse/SummaryNode.hpp` | 34 | ✅ added PR0 |
| `lgr_info::serializeOp` | `opm/io/eclipse/SummaryNode.hpp` | 35-37 | ✅ added PR0 |
| `category_from_keyword()` `case 'L':` | `opm/io/eclipse/SummaryNode.cpp` | ~230 | ✅ added PR0 |
| `SummaryConfigNode lgr_` field, getter, setter | `SummaryConfig.hpp` | ~185 | ✅ added PR0 |
| `operator EclIO::SummaryNode()` forwards `lgr_` | `SummaryConfig.hpp` | 162 | ✅ fixed PR0 |
| `serializeOp()` includes `lgr_` | `SummaryConfig.hpp` | ~178 | ✅ fixed PR0 |
| `operator==` / `operator<` compare `lgr_` | `SummaryConfig.cpp` | 1960-2044 | ✅ fixed PR0 |
| `parseKeywordCategory()` `case 'L':` | `SummaryConfig.cpp` | 1857 | ✅ added PR1 |
| `keywordLW / keywordLC / keywordLB` | `SummaryConfig.cpp` | — | ✅ added PR1 |
| `handleKW()` L-prefix dispatch | `SummaryConfig.cpp` | 1681-1698 | ✅ added PR1 |
| `NOSUMLGR` in `is_processing_instruction()` | `SummaryConfig.cpp` | 337-346 | ✅ added PR1 |
| `noSumLgr_` field + getter + serializeOp | `SummaryConfig.hpp` | — | ✅ added PR1 |
| `handleKW()` takes `GridDims`, NOT `EclipseGrid` | `SummaryConfig.cpp` | 1667 | ✅ confirmed |
| `Well::get_lgr_well_tag()` returns `optional<string>` | `Well.hpp` | 116 | ✅ confirmed |
| `EclipseGrid::getActiveIndexLGR(label, i, j, k)` — 0-based | `EclipseGrid.hpp` | 103 | ✅ confirmed |
| `EclipseGrid::get_all_lgr_labels()` — index 0 = global, skip | `EclipseGrid.hpp` | 136 | ✅ confirmed |
| `SMSPEC write site: SummarySpecification::write()` | `OutputStream.cpp` | 758 | ✅ confirmed |
| `Factory::create()` dispatch chain in Summary.cpp | `Summary.cpp` | 4662 | ✅ confirmed |
| `isBlockValue()` uses global cell index — fails for LGR nodes | `Summary.cpp` | 4781 | ✅ confirmed |
| `funs` map: no LW*/LC* entries — falls through to unknownParameter | `Summary.cpp` | 2549 | ✅ confirmed |
| keyword JSON files: path `000_Eclipse100/L/LOCAL_*_PROBE` | share/keywords/ | — | ✅ confirmed |
| `keyword_list.cmake`: must list every keyword JSON file | share/keywords/ | — | ✅ confirmed |
| `"size": N` in JSON schema = N fixed records; no `"size"` = table | — | — | ✅ confirmed |
| `WELSPECL` sets `lgr_well_tag`; `WELSPECS` does NOT | Schedule/Well | — | ✅ confirmed |
| `Summary.cpp` is in opm-COMMON (not opm-simulators) | `opm/output/eclipse/` | — | ✅ confirmed |
| Eclipse SMSPEC LGR array order: NUMS→LGRS→NUMLX→NUMLY→NUMLZ→UNITS→STARTDAT | FileFormatsRM p.9-11 | — | ✅ confirmed |
| `PaddedOutputString<8>` has no `operator==` or `operator<` | OutputStream | — | ✅ confirmed |

---

## Design Decisions (locked)

1. `keywordLB` does NOT resolve active cell index at parse time — `handleKW()` only has `GridDims`, not `EclipseGrid`. Store raw `lgr_.ijk` (1-based), leave `number()=0`. Evaluator calls `getActiveIndexLGR(name, i-1, j-1, k-1)`.
2. Use `distinguish_well_from_completion()` / `distinguish_connection_from_completion()` in `case 'L':` — defensive; LW*/LC* regex never triggers Completion.
3. `base_keyword()` strips L prefix ("LWOPR"→"WOPR") — file-local static in each .cpp that needs it; NOT in any header.
4. LGRVEC and LGRTIMES are fully out of scope.
5. No `Schedule::wellNames(lgr_name)` overload exists — expand via pattern then filter by `get_lgr_well_tag()`.
6. `noSumLgr_` is serialized (unlike `runSummaryConfig` members) — must survive restarts.

---

## Study Notes (per study)

### Study 1 — operator== / operator< bug (DONE)
Confirmed: both operators in SummaryConfig.cpp miss `lgr_`. Silent dedup drops distinct LGR nodes. Fixed in PR0.

### Study 2 — keywordW expansion algorithm (DONE)
Confirmed: `schedule.wellNames(pattern)` returns all candidates; filter by `get_lgr_well_tag() == lgr_name`. No direct API for wells-in-LGR.

### Study 3 — NOSUMLGR routing (DONE)
Confirmed: absent from `is_processing_instruction()`. Must add to set + handler. Fixed in PR1.

### Study 4 — SPE1CASE1_CARFIN.DATA test deck (DONE)
Found: LGR1 (I=5-6,J=5-6,K=1-3 → 6×6×9 local), LGR2 (I=7-8,J=7-8,K=1-3). Wells PROD(10,10) and INJ(1,1) outside both LGRs. Cannot use for LW*/LC* tests. Can use for LBPR (coords 1..6, 1..6, 1..9).

### Study 5 — Factory::create() insertion point (PENDING)
Need to read `Summary.cpp` lines 4650-4800 to find exact location for `isLgrWellValue()` predicate and `LgrWellValue` evaluator class before writing PR-003.md.
