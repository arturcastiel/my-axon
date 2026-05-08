# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp
shadow-created: 2026-05-04
shadow-updated: 2026-05-07 (rev2 — PR-001 review response)

## Summary
SummaryConfig implementation file (~2400 lines). PR-000 (revised, commit 0066645cd) owns:
`serializationTestObject()`, `operator==`, `operator<` for `SummaryConfigNode`.
PR-001 (`lgr-summaryconfig`) adds on top: `keywordLW`, `keywordLC`, `keywordLB`,
`parseKeywordCategory case 'L':`, `handleKW` LGR dispatch, `NOSUMLGR` handling.
**API HARMONIZATION COMPLETE (2026-05-07):** `keywordLW/LC/LB` updated to new API.

## Key Structures — PR-000 owned (revised API, lgr-summary branch)

### `operator==` (line ~1961)
Compares `lgr_name()` (optional<string>) via `==` in all 3 category groups:
- Well/Node/Group: `(lhs.namedEntity() == rhs.namedEntity()) && (lhs.lgr_name() == rhs.lgr_name())`
- Aquifer/Region/Block: `(lhs.number() == rhs.number()) && (lhs.lgr_name() == rhs.lgr_name())`
- Connection/Completion/Segment: `namedEntity() == && number() == && lgr_name() ==`

### `operator<` (line ~2002) — revised per review (make_tuple, LGR-first)
```cpp
// Well/Node/Group:
return std::make_tuple(lhs.lgr_name(), lhs.namedEntity())
    <  std::make_tuple(rhs.lgr_name(), rhs.namedEntity());

// Aquifer/Region/Block:
return std::make_tuple(lhs.lgr_name(), lhs.number())
    <  std::make_tuple(rhs.lgr_name(), rhs.number());

// Connection/Completion/Segment:
return std::make_tuple(lhs.lgr_name(), lhs.namedEntity(), lhs.number())
    <  std::make_tuple(rhs.lgr_name(), rhs.namedEntity(), rhs.number());
```
`optional<string>::operator<` gives `nullopt < any_value` — global vectors sort before LGR vectors.
LGR identity is the PRIMARY sort key in all groups (reviewer requested, 2026-05-07).

### `serializationTestObject()` (line ~1894)
Sets `result.lgr_name_ = std::string{ "LGR1" }` — exercises optional<string> in roundtrip test.

## Key Structures — PR-001 owned (lgr-summaryconfig branch, API HARMONIZED 2026-05-07)

### `keywordLW` (line ~864)
Filters wells by `get_lgr_well_tag()`. Uses `.lgr_name(lgr_name)` — no lgr_info local.

### `keywordLB` (line ~1094)
Reads only `lgr_name` from deck. Uses `.lgr_name(lgr_name)` — i,j,k dropped (IO-layer concern).

### `keywordLC` (line ~1450)
Filters wells by `get_lgr_well_tag()`. Uses `.lgr_name(lgr_name)` — coords_defaulted/i/j/k dropped.

### `parseKeywordCategory` — `case 'L':`
Routes LW→Well, LC→Connection, LB→Block. No API dependency — unchanged by harmonization.

### `NOSUMLGR` handling
`is_processing_instruction()` + `handleProcessingInstruction()`. No API dependency — unchanged.

## Dependencies
- `SummaryConfig.hpp` — SummaryConfigNode class, lgr_name_ field
- `opm/io/eclipse/SummaryNode.hpp` — EclIO::lgr_info (used only in conversion operator in .hpp)

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-04 | PR0 | `serializationTestObject` sets `lgr_name_ = "LGR1"` |
| 2026-05-04 | PR0 | `operator==` compares `lgr_name()` in all 3 category groups |
| 2026-05-07 | PR0-rev | `operator<` rewritten with `make_tuple` — LGR-first ordering per reviewer (@bska) |
| 2026-05-07 | PR0-rev | Old multi-if chain (`has_value()` mismatch checks) replaced by single-line tuple comparison |
| 2026-05-07 | PR1-study | PR-001 `keywordLW/LC/LB` use old `.lgr(lgr_info{})` setter — stale, must be updated in harmonization |
| 2026-05-07 | PR1-study | PR-001 `operator==`/`operator<`/`serializationTestObject` conflict with PR-000 version — keep PR-000 on rebase |
| 2026-05-07 | PR1-harm | `keywordLW`: dropped `lgr_info` local, now `.lgr_name(lgr_name)` |
| 2026-05-07 | PR1-harm | `keywordLB`: dropped i,j,k reads + `lgr_info`, now `.lgr_name(lgr_name)` |
| 2026-05-07 | PR1-harm | `keywordLC`: dropped coords_defaulted/i/j/k + `lgr_info`, now `.lgr_name(lgr_name)` |
| 2026-05-07 | PR1-review | Block + Connection dispatch arms: braces added to single-statement if/else (style, @bska R10) |
| 2026-05-07 | PR1-review | `keywordLB`: comment added — number_ deferred to follow-up, all same-LGR records share number_=INT_MIN |
| 2026-05-07 | PR1-review | `keywordLC`: same deferred-number_ comment added (@bska R8/R9) |
| 2026-05-07 | PR1-review | `SummaryConfig::operator==`: noSumLgr_ added as 4th comparison term (correctness, was missing) |
