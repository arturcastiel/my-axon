# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/tests/test_SummaryNode.cpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/tests/test_SummaryNode.cpp
shadow-created: 2026-05-05
shadow-updated: 2026-05-05
git-hash: 07c2916828d80cd4b2f9b57097d93a3fd103b0a9758d84f37428c06b522619e2

## Summary
Boost.Test unit tests for `EclIO::SummaryNode`. Two top-level suites: `UniqueKey` (3 cases) and `Category` (5 cases). PR0 added `LGR_category_routing` test case inside the `Category` suite.

## Key Structures

### Suite: `UniqueKey`
- `UniqueKey` — tests all category/type combos produce correct `"KW:name:num"` format
- `InjectedNumberRenderer` — tests custom number renderer lambda via `unique_key(fn)`
- `user_defined` — tests `FU_VAR1` is recognized as UDQ

### Suite: `Category`
Local helpers (anonymous namespace):
- `to_string(Category)` → string — NOT an `operator<<`, just for BOOST_CHECK_MESSAGE diagnostics
- `category(kw)` → calls `SummaryNode::category_from_keyword(kw)`

Test cases:
- `Well` — WOPR/WOPT/WGIR/WWIR → Well; WOPRL → NOT Well
- `Connection` — COPR/COPT/CGIR/CWIR → Connection; COPRL → NOT Connection
- `Completion` — COPRL/COPTL/CGIRL/CWIRL and WOPRL/WOPRL__8/WOPRL123 → Completion; negatives: WOPRLK/CGIR/WKITL__8/WOOOOPRL/WHIRL → NOT Completion
- `LGR_category_routing` — **PR0 ADDITION**:
  ```
  LWOPR  → Well
  LWBHP  → Well
  LCOFR  → Connection
  LCWFR  → Connection
  LBPR   → Block
  LBSOIL → Block
  LLINEARS → Miscellaneous   (second char not W/C/B)
  WOPR   → Well              (regression: unchanged)
  BPR    → Block             (regression: unchanged)
  COFR   → Connection        (regression: unchanged)
  ```

## Dependencies
- `<boost/test/unit_test.hpp>`
- `<opm/io/eclipse/SummaryNode.hpp>`
- `<initializer_list>`, `<stdexcept>`, `<string>`, `<vector>`

## Architecture Role
Unit test coverage for the lowest layer of the LGR pipeline. Verifies `category_from_keyword()` routes LW*/LC*/LB* correctly. These tests run fast (no file I/O, no parser), making them the first regression check for category routing.

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR0 | Added `LGR_category_routing` test case in `BOOST_AUTO_TEST_SUITE(Category)` |
| 2026-05-05 | compile | `Category` enum has NO `operator<<` — `BOOST_CHECK_EQUAL(x, Cat::Well)` fails to compile because Boost needs `operator<<` to format the mismatch message. Must use `BOOST_CHECK(x == Cat::Well)`. |
| 2026-05-05 | compile | Original fix attempt used BOOST_CHECK_EQUAL → compile error → replaced all 10 assertions with BOOST_CHECK(... == ...) |
| 2026-05-05 | test | `LLINEARS` is tested as Miscellaneous — covers the `default: break` path in `case 'L':` for single-char second char not in {W,C,B} |
| 2026-05-05 | test | All 3 tests passed after compile fix: test_SummaryNode PASSED |
