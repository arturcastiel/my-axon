# SHADOW: keyword JSON dir — 000_Eclipse100/L (and N/NOSUMLGR)
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/000_Eclipse100/
shadow-created: 2026-05-05
shadow-updated: 2026-05-05

## Summary
Keyword description JSON files. The OPM parser needs one file per keyword (or regex group) to
know item count and types. Each file lives under a single-letter subdirectory matching the
keyword's first character. NOSUMLGR goes in N/.

## Existing relevant files

### W/WELL_PROBE
- `deck_name_regex: "WU.+|..."` covers WU* wildcards
- `size: 1` — one record per entry
- `items: [{ "name": "WELLS", "value_type": "STRING", "size_type": "ALL" }]`
- Explicit `deck_names` list for all W* summary vectors
- Template for LOCAL_WELL_PROBE (add LGR_NAME as item 0, WELLS as item 1)

### C/CONNECTION_PROBE
- No `size` key — unlimited records
- `items: [WELL(STRING), I(INT), J(INT), K(INT)]`
- **CRITICAL**: already has `LCOFRU`, `LCWFRU`, `LCGFRU` in explicit `deck_names` — uses OLD schema (no LGR_NAME). Do NOT include these in LOCAL_CONNECTION_PROBE.
- Template for LOCAL_CONNECTION_PROBE (add LGR_NAME as item 0)

### B/BLOCK_PROBE
- No `size` key
- `items: [I(INT), J(INT), K(INT)]`
- **CRITICAL**: already has `LBPR_X`, `LBSCN_X`, `LBHD_X`, `LBHDF_X`, `LBCTRA_X` in explicit `deck_names`. Do NOT include these in LOCAL_BLOCK_PROBE.
- Template for LOCAL_BLOCK_PROBE (add LGR_NAME as item 0)

### N/NARROW
- Minimal format: `{ "name": "NARROW", "sections": ["SUMMARY"] }` — no items
- Template for NOSUMLGR

## New files to create (PR01)

### L/LOCAL_WELL_PROBE
```json
{
  "name": "LOCAL_WELL_PROBE",
  "sections": ["SUMMARY"],
  "comment": "LGR well-level summary vectors (LW*). Item 0 is LGR_NAME; item 1 is WELLS.",
  "deck_name_regex": "LW.+",
  "size": 1,
  "items": [
    { "name": "LGR_NAME", "value_type": "STRING" },
    { "name": "WELLS",    "value_type": "STRING", "size_type": "ALL" }
  ]
}
```

### L/LOCAL_CONNECTION_PROBE
```json
{
  "name": "LOCAL_CONNECTION_PROBE",
  "sections": ["SUMMARY"],
  "comment": "LGR connection-level summary vectors (LC*). Item 0 is LGR_NAME.",
  "deck_name_regex": "LC.+",
  "items": [
    { "name": "LGR_NAME", "value_type": "STRING" },
    { "name": "WELL",     "value_type": "STRING" },
    { "name": "I",        "value_type": "INT"    },
    { "name": "J",        "value_type": "INT"    },
    { "name": "K",        "value_type": "INT"    }
  ]
}
```
NOTE: LCOFRU/LCWFRU/LCGFRU are already claimed by CONNECTION_PROBE explicit deck_names.
The `LC.+` regex will NOT override explicit deck_names — those three are safe.

### L/LOCAL_BLOCK_PROBE
```json
{
  "name": "LOCAL_BLOCK_PROBE",
  "sections": ["SUMMARY"],
  "comment": "LGR block-level summary vectors (LB*). Item 0 is LGR_NAME.",
  "deck_name_regex": "LB.+",
  "items": [
    { "name": "LGR_NAME", "value_type": "STRING" },
    { "name": "I",        "value_type": "INT"    },
    { "name": "J",        "value_type": "INT"    },
    { "name": "K",        "value_type": "INT"    }
  ]
}
```
NOTE: LBPR_X/LBSCN_X/LBHD_X/LBHDF_X/LBCTRA_X already claimed by BLOCK_PROBE explicit deck_names.

### N/NOSUMLGR
```json
{
  "name": "NOSUMLGR",
  "sections": ["SUMMARY"]
}
```

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR01 study | LOCAL_WELL_PROBE, LOCAL_CONNECTION_PROBE, LOCAL_BLOCK_PROBE, NOSUMLGR all absent — must be created |
| 2026-05-05 | PR01 study | `deck_name_regex` takes precedence for LW*/LC*/LB* except where explicit deck_names in other files override |
| 2026-05-05 | PR01 study | `size: 1` needed in LOCAL_WELL_PROBE to match WELL_PROBE pattern (one record, multi-item WELLS column) |
| 2026-05-05 | PR01 study | No `size` key in LOCAL_CONNECTION_PROBE and LOCAL_BLOCK_PROBE — unlimited records |
