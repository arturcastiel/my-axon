# SHADOW: NOSUMLGR keyword JSON
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/000_Eclipse100/N/NOSUMLGR
shadow-created: 2026-05-05
shadow-updated: 2026-05-05
pr: PR01

## Summary
Keyword description file for the `NOSUMLGR` processing instruction. No items — keyword
presence alone triggers the flag. Identical structure to NARROW and RUNSUM.

## Content
```json
{
  "name": "NOSUMLGR",
  "sections": ["SUMMARY"]
}
```

## Key Facts
- No `items` — keyword presence is the signal, no parameters
- `is_processing_instruction()` now includes NOSUMLGR → routes to `handleProcessingInstruction()`
- `handleProcessingInstruction("NOSUMLGR")` sets `SummaryConfig::noSumLgr_ = true`
- Without this file: parser would reject the keyword or route to keywordMISC (silent ignore)
- Phase 2 behaviour (ROOT.LGR suppression) deferred — flag stored and serialized only

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR01 ✓ | Created — absent before PR01; NOSUMLGR silently dropped without it |
| 2026-05-05 | design | Must also be in is_processing_instruction() set in SummaryConfig.cpp, otherwise handleKW routes it to keywordMISC |
