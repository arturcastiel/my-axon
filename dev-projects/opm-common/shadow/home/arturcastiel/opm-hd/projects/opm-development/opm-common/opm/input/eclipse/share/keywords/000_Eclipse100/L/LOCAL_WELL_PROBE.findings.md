# SHADOW: LOCAL_WELL_PROBE keyword JSON
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_WELL_PROBE
shadow-created: 2026-05-05
shadow-updated: 2026-05-05
pr: PR01

## Summary
Keyword description file for the `LW*` family of LGR well-level summary vectors.
Instructs the OPM parser how many items each deck record contains and what types they are.

## Content (current — fixed)
```json
{
  "name": "LOCAL_WELL_PROBE",
  "sections": ["SUMMARY"],
  "deck_name_regex": "LW.+",
  "items": [
    { "name": "LGR_NAME", "value_type": "STRING" },
    { "name": "WELLS",    "value_type": "STRING", "size_type": "ALL" }
  ]
}
```

## Key Facts
- `deck_name_regex: "LW.+"` — matches all LW* keywords (LWOPR, LWOPT, LWOIR, etc.)
- NO `"size"` field — table format; one record per LGR/well pair, block terminated by `/`
- `"size": 1` would mean a single fixed record → rejects multi-record table → **was the Jenkins bug**
- `LOCAL_CONNECTION_PROBE` and `LOCAL_BLOCK_PROBE` were already correct (no `"size"` field)
- Item 0 = LGR_NAME (string), Item 1 = WELLS (string, size_type ALL)
- `keywordLW()` reads: `record.getItem(0)` = LGR_NAME, `record.getItem(1)` = WELLS
- `keywordLW` also filters by `well.get_lgr_well_tag() == lgr_name` — well must be declared via `WELSPECL` (not `WELSPECS`) for nodes to be emitted

## WELSPECL record format (confirmed from reference deck)
`WELL GROUP LGR HEAD_I HEAD_J REF_DEPTH PHASE ...`
Example: `'INJ'  'G1'  'LGR1'  2  2  8335  'GAS' /`
- `WELSPECL` calls `handleWELSPECS` internally (registers well) then sets `lgr_well_tag`
- `COMPDATL` is NOT needed for `keywordLW` node generation — only `WELSPECL` required

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR01 ✓ | Created — absent before PR01; LW* keywords were silently routed to Miscellaneous |
| 2026-05-05 | design | Initially added `"size": 1` incorrectly, believing it matched WELL_PROBE pattern |
| 2026-05-06 | Jenkins failure | `"size": 1` caused parser to reject multi-record LWBHP blocks → `OpmInputError: Extra line starting with '/'` |
| 2026-05-06 | fix | Removed `"size": 1` — now table format matching LOCAL_CONNECTION_PROBE/LOCAL_BLOCK_PROBE |
| 2026-05-06 | inline tests | `LGR_Schema_Inline` suite added to `test_SummaryConfigNode.cpp` — would have caught this immediately |
