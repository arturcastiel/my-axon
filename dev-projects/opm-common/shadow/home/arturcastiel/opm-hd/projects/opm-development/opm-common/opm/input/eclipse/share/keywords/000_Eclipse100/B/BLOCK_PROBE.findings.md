# SHADOW: BLOCK_PROBE keyword JSON
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/000_Eclipse100/B/BLOCK_PROBE
shadow-created: 2026-05-07
shadow-updated: 2026-05-07
pr: PR01-review

## Summary
Keyword description file for the global block-level summary vectors (`B*` family).
Processed by `keywordB()` in SummaryConfig.cpp. Item set: `I, J, K` (global 1-based indices, no WELL or LGR_NAME).

## Key Facts
- `deck_name_regex`: large alternation covering `BU.+`, `BTIPF.+`, etc. — explicit `deck_names` are the primary route
- No `size` key — unlimited records (table format)
- Items: `I` (INT), `J` (INT), `K` (INT) — global grid indices only, no WELL, no LGR_NAME
- `keywordB()` validates ijk against `GridDims` (global dims) — invalid cells are warned and skipped
- Dispatch: SummaryConfig routes `B`-prefix to `keywordB`; `LB`-prefix to `keywordLB` (different keyword)

## deck_names — LGR-related (post-PR-001-review state)
- `LBCTRA_X`, `LBHDF_X`, `LBHD_X`, `LBPR_X`, `LBSCN_X` — **REMOVED** in PR-001 review response
  - These had `LB` prefix but required a different item format (no LGR_NAME in BLOCK_PROBE items)
  - Reviewer @bska flagged (R4): format mismatch — they are unsupported and should not be in any probe
  - After removal, `LB[A-Z]+` regex in LOCAL_BLOCK_PROBE has no ambiguous dual-match for these names

## deck_names — notable global _X variants (still present)
- `BCTRA_X`, `BHDF_X`, `BHD_X`, `BPR_X`, `BSCN_X` — global tracer/pressure variants, remain in BLOCK_PROBE

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-07 | PR1-review | First study — file touched to remove LB*_X entries (@bska R4) |
| 2026-05-07 | PR1-review | Items are `I, J, K` only — no LGR_NAME; LB*_X do not fit this schema |
| 2026-05-07 | PR1-review | After removal, `LB[A-Z]+` in LOCAL_BLOCK_PROBE has clean, unambiguous coverage |
