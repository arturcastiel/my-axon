# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/tests/test_SummaryConfigNode.cpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/tests/test_SummaryConfigNode.cpp
shadow-created: 2026-05-05
shadow-updated: 2026-05-07 (rev2 — PR-001 review response)

## Summary
Boost.Test unit tests for `SummaryConfigNode`. File is modified by both PR-000 and PR-001.

**Current state on `lgr-summary` (PR-000, commit 0066645cd):** 2 top-level suites:
- `ParseKeywords` (pre-existing) — keyword type parsing
- `LGR` — 6 test cases (3 original + 3 added per reviewer @bska, 2026-05-07)

**State on `lgr-summaryconfig` (PR-001, stale API):** Many additional suites added using old `.lgr()` API.
These must be updated to `lgr_name()` in harmonization.

## Suite: `ParseKeywords` (pre-existing)
Sub-suites: `Type::Total`, `Type::Rate`, `Type::Pressure`, `Type::Ratio`, `Type::ProdIndex`,
`Type::Count`, `Type::Undefined`. Named test cases for specific keywords.

## Suite: `LGR` — PR-000 owned (6 cases, `lgr-summary` canonical)

### `lgr_field_roundtrip`
Getter/setter roundtrip for Well and Block; conversion to `EclIO::SummaryNode` preserves `lgr->name`;
non-LGR node stays nullopt on both sides.

### `lgr_dedup_distinct_lgrs`
`LWOPR+LGR1+PROD1 != LWOPR+LGR2+PROD1`. Strict weak ordering holds. Same LGR → equal.
Regression guard for silent dedup drop in `std::set<SummaryConfigNode>`.

### `lgr_no_lgr_sorts_before_lgr`
Global `WOPR+PROD1` sorts before `WOPR+LGR1+PROD1`. Null LGR < any named LGR. (Well category)

### `lgr_operator_equal_block_and_connection` ← added per review (2026-05-07)
`operator==` for Block (same number+LGR = equal; different LGR = not equal) and Connection.

### `lgr_operator_less_lgr_before_entity` ← added per review (2026-05-07)
LGR sorts before entity: `LGR1+PROD_Z < LGR2+PROD_A` for Well, Block, Connection.
Key behavioral test for the revised `make_tuple` ordering.

### `lgr_operator_less_null_before_set` ← added per review (2026-05-07)
Null LGR < named LGR for Block and Connection. Complements `lgr_no_lgr_sorts_before_lgr` (Well only).

## Suites added by PR-001 (lgr-summaryconfig, API HARMONIZED 2026-05-07)

### `ParseKeywords/Category`
`LWOPR_is_Well`, `LCOFR_is_Connection`, `LBPR_is_Block`, `WOPR_is_Well`, `unknown_L_is_Misc`.
Tests `parseKeywordCategory` routing. No API dependency — survived rebase unchanged.

### `NoSumLgr`
`default_is_false`, `serialization_test_object_sets_flag`, `deck_sets_flag`.
Tests `noSumLgr_` field. `deck_sets_flag` added in PR-001 review response — parses inline minimal deck
with `NOSUMLGR` keyword, verifies `cfg.noSumLgr() == true`. Uses raw inline deck (not
`parseSummarySection`) to avoid anonymous-namespace scope issue.

### `LGR_SummaryConfig_Deck` (6 integration tests) — API UPDATED
All `.lgr()` → `.lgr_name()`, all `->ijk[N]` assertions dropped. `lgr_2lgr_lb_dedup_by_lgr_name`
rewritten to count by LGR name only (ijk is IO-layer, not config-layer).

### `LGR_Schema_Inline` (5 inline WELSPECL tests) — API UPDATED
All `.lgr()` → `.lgr_name()`, all ijk assertions dropped. Added `BOOST_REQUIRE` guards before
dereference in `lw_multi_record_produces_two_nodes` and `lw_regex_matches_lwstat`.

### PR-001 old `LGR` suite (3 cases) — DROPPED on rebase
`lgr_field_roundtrip`, `lgr_dedup_distinct_lgrs`, `lgr_no_lgr_sorts_before_lgr` — superseded
by PR-000's 6-case suite. Dropped cleanly during `git rebase --skip` of old PR-000 commit.

## Dependencies
- `<boost/test/unit_test.hpp>`
- `<opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.hpp>`
- `<opm/io/eclipse/SummaryNode.hpp>`
- Test-file-local `operator<<` for `SummaryNode::Type` (for BOOST_CHECK_EQUAL on Type)

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR0 | Added `LGR` suite with 3 test cases using old `.lgr(lgr_info{})` API |
| 2026-05-07 | PR0-rev | LGR suite expanded to 6 test cases; updated to `lgr_name()` API |
| 2026-05-07 | PR0-rev | 3 new cases: `lgr_operator_equal_block_and_connection`, `lgr_operator_less_lgr_before_entity`, `lgr_operator_less_null_before_set` |
| 2026-05-07 | PR1-study | PR-001 adds 4 more suites but all Deck/Schema tests use stale `.lgr()` API + ijk assertions |
| 2026-05-07 | PR1-study | PR-001's 3-case LGR suite must be dropped on rebase — superseded by PR-000's 6-case suite |
| 2026-05-07 | PR1-harm | All `.lgr()` → `.lgr_name()`, all `->ijk[N]` dropped in LGR_SummaryConfig_Deck + LGR_Schema_Inline |
| 2026-05-07 | PR1-harm | `lgr_2lgr_lb_dedup_by_lgr_name` rewritten: count by LGR name only (no ijk at config layer) |
| 2026-05-07 | PR1-harm | Added BOOST_REQUIRE guards before optional dereference in multi-record tests |
| 2026-05-07 | design | `Category` enum has no `operator<<` — use `BOOST_CHECK(x == Cat::Y)` not `BOOST_CHECK_EQUAL` |
| 2026-05-07 | design | `Type` enum has local `operator<<` in test file — `BOOST_CHECK_EQUAL` works for Type only |
| 2026-05-07 | PR1-review | All unconstrained `std::find_if/any_of/count_if` → `std::ranges::*` (@bska R2/R11) |
| 2026-05-07 | PR1-review | `parseSummarySection` deck_str converted to `R"(...)"` raw string literal (@bska R3/R12) |
| 2026-05-07 | PR1-review | `lc_multi_record_produces_nodes` size assertion: 2→1 (deferred number_ → dedup collapses same-LGR) |
| 2026-05-07 | PR1-review | `lb_multi_record_produces_nodes` size assertion: 2→1 (same reason) |
| 2026-05-07 | PR1-review | `NoSumLgr/deck_sets_flag` test added — verifies NOSUMLGR deck keyword sets noSumLgr_ flag |
