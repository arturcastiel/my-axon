# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp
shadow-created: 2026-05-04
shadow-updated: 2026-05-04
git-hash: fc3a69a8e734efff0f226000be7080913ad0d26f47843b71d7117b58a06366af

## Summary
SummaryConfig implementation file. PR-000 changes: (1) `serializationTestObject()` sets `lgr_name_ = "LGR1"`; (2) `operator==` compares `lgr_name()` in all 3 category groups; (3) `operator<` uses null-check + string compare on `lgr_name_`. `keywordLW`/`keywordLC`/`keywordLB` are NOT in this PR — those are PR-001 (lgr-summaryconfig branch).

## Key Structures

### `operator==`
Compares `lgr_name()` (optional<string>) via plain `==` in all 3 category groups (Well/Block/Connection).

### `operator<`
Tiebreaker chain: entity name → `lgr_name_.has_value()` mismatch (no-lgr sorts before lgr) → `*lgr_name()` string compare. No ijk.

### `serializationTestObject()`
Sets `lgr_name_ = std::string{ "LGR1" }` so the new field is exercised by the serialization roundtrip test.

## Dependencies
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.hpp` — SummaryConfigNode class
- `opm/io/eclipse/SummaryNode.hpp` — EclIO::lgr_info (used in conversion operator, not in .cpp directly)

## Architecture Role
Config layer parse/compare/sort logic. Does NOT implement keywordLW/LC/LB (that's PR-001).
