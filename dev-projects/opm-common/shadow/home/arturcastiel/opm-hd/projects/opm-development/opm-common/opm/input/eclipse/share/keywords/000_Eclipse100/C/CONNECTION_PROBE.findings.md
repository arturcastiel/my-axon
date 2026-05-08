# SHADOW: CONNECTION_PROBE keyword JSON
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/000_Eclipse100/C/CONNECTION_PROBE
shadow-created: 2026-05-07
shadow-updated: 2026-05-07
pr: PR01-review

## Summary
Keyword description file for the global connection-level summary vectors (`C*` family).
Processed by `connectionKeyword()` in SummaryConfig.cpp. Item set: `WELL, I, J, K` — no LGR_NAME.

## Key Facts
- `deck_name_regex`: large alternation covering `CU.+`, `CTFR.+`, `CTPR.+`, etc.
- No `size` key — unlimited records (table format)
- Items: `WELL` (STRING), `I` (INT), `J` (INT), `K` (INT) — global connection format, no LGR_NAME
- `connectionKeyword()` resolves well names, applies defaulting for `*`-patterns
- Dispatch: SummaryConfig routes `C`-prefix to `connectionKeyword`; `LC`-prefix to `keywordLC` (different keyword)

## deck_names — LGR-related (post-PR-001-review state)
- `LCOFRU`, `LCWFRU`, `LCGFRU` — **REMOVED** in PR-001 review response
  - These had `LC` prefix but were placed in CONNECTION_PROBE's item format (WELL, I, J, K — no LGR_NAME)
  - Reviewer @bska flagged (R6): format mismatch — they don't fit CONNECTION_PROBE's item set
  - After removal, `LC[A-Z]+` regex in LOCAL_CONNECTION_PROBE has no ambiguous dual-match for these names
  - These vectors are currently unsupported and unplaced — follow-up work required if needed

## deck_names — global *FRU variants (still present)
- `COFRU`, `CWFRU`, `CGFRU` — global fractional-flow variants, remain in CONNECTION_PROBE (no `L` prefix, correct schema)

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-07 | PR1-review | First study — file touched to remove LCOFRU/LCWFRU/LCGFRU entries (@bska R6) |
| 2026-05-07 | PR1-review | Items are `WELL, I, J, K` — no LGR_NAME; LC*FRU do not fit this schema |
| 2026-05-07 | PR1-review | Global COFRU/CWFRU/CGFRU (no L prefix) remain — correct placement, correct schema |
| 2026-05-07 | PR1-review | After removal, `LC[A-Z]+` in LOCAL_CONNECTION_PROBE has clean, unambiguous coverage |
