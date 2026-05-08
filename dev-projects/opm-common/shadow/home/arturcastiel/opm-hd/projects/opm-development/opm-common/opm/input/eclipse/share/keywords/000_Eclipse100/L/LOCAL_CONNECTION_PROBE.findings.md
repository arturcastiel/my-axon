# SHADOW: LOCAL_CONNECTION_PROBE keyword JSON
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_CONNECTION_PROBE
shadow-created: 2026-05-05
shadow-updated: 2026-05-07
pr: PR01

## Summary
Keyword description file for the `LC*` family of LGR connection-level summary vectors.

## Content
```json
{
  "name": "LOCAL_CONNECTION_PROBE",
  "sections": ["SUMMARY"],
  "deck_name_regex": "LC[A-Z]+",
  "items": [
    { "name": "LGR_NAME", "value_type": "STRING" },
    { "name": "WELL",     "value_type": "STRING" },
    { "name": "I",        "value_type": "INT"    },
    { "name": "J",        "value_type": "INT"    },
    { "name": "K",        "value_type": "INT"    }
  ]
}
```

## Key Facts
- `deck_name_regex: "LC[A-Z]+"` — matches LC* keywords with uppercase letters only; excludes `_X` and numeric suffix patterns
- No `size` key — unlimited records
- Item 0 = LGR_NAME, Item 1 = WELL, Items 2/3/4 = I/J/K (1-based local coords, all defaultable)
- `keywordLC()` reads: item(0)=LGR_NAME, item(1)=WELL; i/j/k dropped (IO-layer concern, PR-002)
- LCOFRU, LCWFRU, LCGFRU removed from CONNECTION_PROBE (PR-001 review response @bska R6) — unsupported format

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR01 ✓ | Created — absent before PR01 |
| 2026-05-05 | design | LCOFRU/LCWFRU/LCGFRU excluded — already in CONNECTION_PROBE with old schema (no LGR_NAME item 0) |
| 2026-05-05 | design | No size key = unlimited records (matches CONNECTION_PROBE pattern) |
| 2026-05-07 | PR1-review | regex tightened to `LC[A-Z]+` (@bska R1/R4) — excludes `_X` and numeric suffix patterns |
| 2026-05-07 | PR1-review | stale comment referencing LCOFRU/LCWFRU/LCGFRU removed from JSON comment field (@bska R7) |
| 2026-05-07 | PR1-review | LCOFRU/LCWFRU/LCGFRU removed from CONNECTION_PROBE (see CONNECTION_PROBE shadow) — no dual-match risk |
