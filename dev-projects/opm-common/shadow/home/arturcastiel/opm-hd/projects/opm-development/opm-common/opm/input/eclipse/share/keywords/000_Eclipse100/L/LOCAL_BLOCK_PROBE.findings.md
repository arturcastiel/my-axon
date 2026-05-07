# SHADOW: LOCAL_BLOCK_PROBE keyword JSON
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_BLOCK_PROBE
shadow-created: 2026-05-05
shadow-updated: 2026-05-05
pr: PR01

## Summary
Keyword description file for the `LB*` family of LGR block-level summary vectors.

## Content
```json
{
  "name": "LOCAL_BLOCK_PROBE",
  "sections": ["SUMMARY"],
  "deck_name_regex": "LB.+",
  "items": [
    { "name": "LGR_NAME", "value_type": "STRING" },
    { "name": "I",        "value_type": "INT"    },
    { "name": "J",        "value_type": "INT"    },
    { "name": "K",        "value_type": "INT"    }
  ]
}
```

## Key Facts
- `deck_name_regex: "LB.+"` — matches all LB* keywords (LBPR, LBSAT, etc.)
- No `size` key — unlimited records
- Item 0 = LGR_NAME, Items 1/2/3 = I/J/K (1-based local coords, no wildcards for LB*)
- `keywordLB()` reads: item(0)=LGR_NAME, item(1/2/3)=i/j/k as raw 1-based deck ints
- No bounds validation — GridDims only has global dims, not per-LGR dims
- **CRITICAL**: LBPR_X, LBSCN_X, LBHD_X, LBHDF_X, LBCTRA_X already in BLOCK_PROBE explicit
  deck_names (old schema, no LGR_NAME). Not captured by `LB.+` regex.

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR01 ✓ | Created — absent before PR01 |
| 2026-05-05 | design | LBPR_X and _X variants excluded — already in BLOCK_PROBE with old schema |
| 2026-05-05 | design | keywordLB stores raw 1-based local coords; no global index — evaluator resolves via getActiveIndexLGR() |
