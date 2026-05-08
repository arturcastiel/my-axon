# SHADOW: LOCAL_BLOCK_PROBE keyword JSON
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_BLOCK_PROBE
shadow-created: 2026-05-05
shadow-updated: 2026-05-07
pr: PR01

## Summary
Keyword description file for the `LB*` family of LGR block-level summary vectors.

## Content
```json
{
  "name": "LOCAL_BLOCK_PROBE",
  "sections": ["SUMMARY"],
  "deck_name_regex": "LB[A-Z]+",
  "items": [
    { "name": "LGR_NAME", "value_type": "STRING" },
    { "name": "I",        "value_type": "INT"    },
    { "name": "J",        "value_type": "INT"    },
    { "name": "K",        "value_type": "INT"    }
  ]
}
```

## Key Facts
- `deck_name_regex: "LB[A-Z]+"` — matches LB* keywords with uppercase letters only; excludes `_X` and numeric suffix patterns
- No `size` key — unlimited records
- Item 0 = LGR_NAME, Items 1/2/3 = I/J/K (1-based local coords, no wildcards for LB*)
- `keywordLB()` reads: item(0)=LGR_NAME only; i/j/k dropped (IO-layer concern, PR-002)
- No bounds validation — GridDims only has global dims, not per-LGR dims
- LBPR_X, LBSCN_X, LBHD_X, LBHDF_X, LBCTRA_X removed from BLOCK_PROBE (PR-001 review response @bska R4)
  and will not match `LB[A-Z]+` — unsupported format

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR01 ✓ | Created — absent before PR01 |
| 2026-05-05 | design | LBPR_X and _X variants excluded — already in BLOCK_PROBE with old schema |
| 2026-05-05 | design | keywordLB stores raw 1-based local coords; no global index — evaluator resolves via getActiveIndexLGR() |
| 2026-05-07 | PR1-review | regex tightened to `LB[A-Z]+` (@bska R5) — excludes `_X` and numeric suffix patterns |
| 2026-05-07 | PR1-review | stale comment referencing _X keywords removed from JSON comment field (@bska R5) |
| 2026-05-07 | PR1-review | LB*_X entries removed from BLOCK_PROBE (see BLOCK_PROBE shadow) — no dual-match risk |
