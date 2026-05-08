# PR-001 — GitHub PR Description

**Title:** SummaryConfig: parse LW*/LC*/LB* LGR summary vectors and NOSUMLGR

---

## Summary

Adds `SummaryConfig`-layer support for Local Grid Refinement (LGR) summary
vectors: `LW*` (well-level), `LC*` (connection-level), `LB*` (block-level),
and the `NOSUMLGR` processing instruction.

This is the second step of the LGR summary pipeline. The first step —
`SummaryConfigNode::lgr_name_` field, `operator==`/`operator<` extensions,
and `category_from_keyword` `'L'` routing — was merged in PR #5140.

## Depends on

PR #5140 (merged into master, commit `0066645cd`).

## Commits

| Commit | Description |
|--------|-------------|
| `56cc3cb89` | keyword: add parser schema for LGR summary vectors and NOSUMLGR |
| `dbfeddd15` | SummaryConfig: parse LW*/LC*/LB* keywords and handle NOSUMLGR |
| `f3f0c7a63` | tests: SummaryConfig LGR keyword parsing and NOSUMLGR |

## Changes

### Keyword schemas (`opm/input/eclipse/share/keywords/`)

Four new keyword definition files, registered in `keyword_list.cmake`:

- **`LOCAL_WELL_PROBE`** — matches `LW.+`. Table format (no `"size"` field).
  Two items per record: `LGR_NAME` (string) and `WELLS` (string, `size_type: ALL`
  for wildcard expansion).

- **`LOCAL_CONNECTION_PROBE`** — matches `LC.+`. Table format. Five items:
  `LGR_NAME`, `WELL`, `I`, `J`, `K`. Coordinates default to 0 and are an
  IO-layer concern (not stored in `SummaryConfigNode`).

- **`LOCAL_BLOCK_PROBE`** — matches `LB.+`. Table format. Four items:
  `LGR_NAME`, `I`, `J`, `K`.

- **`NOSUMLGR`** — no items. Processing instruction that suppresses LGR
  summary output.

### `SummaryConfig.cpp`

- `parseKeywordCategory`: new `case 'L'` routes `LW*` → `Well`,
  `LC*` → `Connection`, `LB*` → `Block`; other `L*` fall through to
  `Miscellaneous`.
- `keywordLW`: builds one `Well` `SummaryConfigNode` per WELSPECL well whose
  `get_lgr_well_tag()` matches the record's LGR name.
- `keywordLC`: same pattern, `Connection` category.
- `keywordLB`: builds one `Block` node per deck record, keyed by LGR name.
  Cell coordinates (I, J, K) are parsed by the deck but not stored here —
  they are an IO-layer concern resolved from SMSPEC arrays.
- `handleKW`: dispatch arm for `LGR` category.
- `is_processing_instruction` / `handleProcessingInstruction`: `NOSUMLGR`
  support.

### `SummaryConfig.hpp`

- `noSumLgr_` bool field, `noSumLgr()` getter, `serializeOp` entry.

### Tests (`tests/`)

**`test_SummaryConfigNode.cpp`** — four suites:

- `ParseKeywords/Category`: verifies `parseKeywordCategory` routing for
  `LWOPR`, `LCOFR`, `LBPR`, `WOPR`, and an unknown `L*` keyword.
- `NoSumLgr`: `default_is_false`, `serialization_test_object_sets_flag`.
- `LGR_SummaryConfig_Deck` (6 tests): integration tests against `.DATA` files
  covering LW*/LC*/LB* node population, global-well isolation, and dedup
  behaviour across multiple LGRs.
- `LGR_Schema_Inline` (5 tests): inline-deck tests — schema format, table
  expansion, well-tag matching, and keyword regex coverage. No external files.

**New test decks:**
- `tests/LGR-WELL-3X3-1LGR.DATA` — 3×3 global grid, one LGR; INJ in LGR1,
  PROD in global grid.
- `tests/LGR-WELL-3X3-2LGR.DATA` — same grid, two LGRs; dedup stress test.

## Testing

```
ctest -R "^(SummaryNode|SummaryConfigTests|Serialization)$" --output-on-failure
```

All 3 tests pass (10.8 s total).
