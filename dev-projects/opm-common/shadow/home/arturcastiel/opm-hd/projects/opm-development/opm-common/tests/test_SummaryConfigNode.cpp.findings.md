# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/tests/test_SummaryConfigNode.cpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/tests/test_SummaryConfigNode.cpp
shadow-created: 2026-05-05
shadow-updated: 2026-05-05
git-hash: 49a2843f0f86bb16cfc10f2d88479ab5d01ea20d01969f02989cbd384e31b781

## Summary
Boost.Test unit tests for `SummaryConfigNode` (mid-layer). Two top-level suites: `ParseKeywords` (keyword type parsing — pre-existing) and `LGR` (PR0 addition — 3 cases testing `lgr_` field roundtrip, dedup safety, and sort ordering).

## Key Structures

### Test-file-local `operator<<` for `SummaryNode::Type`
Defined in `namespace Opm::EclIO` at file scope so `BOOST_CHECK_EQUAL` works for `Type` comparisons. Note: this is for `Type`, NOT `Category` — `Category` still has no `operator<<`.

### Suite: `ParseKeywords` (pre-existing)
Sub-suites: `Type::Total`, `Type::Rate`, `Type::Pressure`, `Type::Ratio`, `Type::ProdIndex`, `Type::Count`, `Type::Undefined`
Each has named test cases for specific keywords (BOPT, COPT, FOPT, GOPT, ROPT, WOPT, CGMITL…).

### Suite: `LGR` — **PR0 ADDITION**

#### `lgr_field_roundtrip`
- Builds `SummaryConfigNode("LWOPR", Cat::Well, loc)`, calls `.lgr(lgr_info{"LGR1", {0,0,0}})`
- Checks `lgr().has_value()`, name, ijk[0-2]
- Converts to `EclIO::SummaryNode` via `operator EclIO::SummaryNode()` — verifies `sn.lgr` preserved
- Verifies non-LGR node (`"WOPR"`) has `lgr()` == nullopt, converts to `sn2.lgr` == nullopt
- Tests `LBPR` with ijk `{2,3,4}` — verifies ijk preserved after conversion

#### `lgr_dedup_distinct_lgrs`
- Creates two `LWOPR/PROD1` nodes — LGR1 vs LGR2
- ASSERT `n1 != n2` — prevents silent dedup in `std::set<SummaryConfigNode>`
- ASSERT strict weak ordering holds: `n1 < n2 || n2 < n1`
- Creates `n3` = LGR1 (same as n1) — ASSERT `n1 == n3` and `!(n1 < n3) && !(n3 < n1)`

#### `lgr_no_lgr_sorts_before_lgr`
- `global` = `WOPR/PROD1` with no lgr
- `lgr_node` = `WOPR/PROD1` with LGR1
- ASSERT `global < lgr_node`
- ASSERT `!(lgr_node < global)` — asymmetry check

## Dependencies
- `<boost/test/unit_test.hpp>`
- `<opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.hpp>`
- `<opm/io/eclipse/SummaryNode.hpp>`
- `<initializer_list>`, `<ostream>`, `<stdexcept>`, `<string>`, `<type_traits>`, `<vector>`

## Architecture Role
Regression guard for the mid-layer LGR data path. Tests that `lgr_` field survives: (1) getter/setter roundtrip, (2) conversion to `EclIO::SummaryNode`, (3) equality comparison, (4) `operator<` sort order. Critical for preventing silent dedup bugs in `std::set<SummaryConfigNode>` when two nodes have same keyword+entity but different LGRs.

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR0 | Added `BOOST_AUTO_TEST_SUITE(LGR)` after end of `ParseKeywords` suite |
| 2026-05-05 | PR0 | 3 test cases: `lgr_field_roundtrip`, `lgr_dedup_distinct_lgrs`, `lgr_no_lgr_sorts_before_lgr` |
| 2026-05-05 | test | All 3 tests PASSED: test_SummaryConfigNode |
| 2026-05-05 | design | `operator==` in SummaryConfig.cpp compares `lgr()` for Well/Block/Connection categories; Group/Field/etc skip lgr comparison (no LGR semantics for those categories) |
| 2026-05-05 | design | `operator<` tiebreaker chain: keyword → category → type → name → number → fip_region → lgr.has_value() → lgr.name → lgr.ijk[0] → lgr.ijk[1] → lgr.ijk[2] |
| 2026-05-05 | PR01-note | Integration tests for LGR_Summary (reading actual .DATA files) belong to PR01 — they require `parseKeywordCategory case 'L':` and `keywordLW/LC/LB` functions not yet implemented |
