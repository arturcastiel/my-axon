# Shadow: opm-tests/summary/results/LGR-WELL-3X3-2LGR.FSMSPEC
project: opm-common
source: Eclipse reference simulator output (ASCII formatted)
indexed: 2026-05-05

## Summary
Eclipse reference SMSPEC for the 2-LGR test case. 59 total vectors:
21 non-LGR (TIME, YEARS, F*, W*, BPR, BGSAT) + 38 LGR (19 per LGR: LW*/LC*/LB* for INJ in LGR1 and PROD in LGR2).
Ground truth for 2-LGR OPM SMSPEC output. Adds LGRNAMES/LGRVEC/LGRTIMES with 2 entries.

## Array Inventory
| Array | Count | Notes |
|---|---|---|
| INTEHEAD | 2 ints | `{2, 100}` |
| RESTART | 9 chars | all blank |
| DIMENS | 6 ints | `{59, 3, 3, 1, 0, -1}` |
| KEYWORDS | 59 chars | |
| WGNAMES | 59 chars | |
| NUMS | 59 ints | |
| LGRS | 59 chars | idx 0–20 blank; idx 21–58 alternating 'LGR1    '/'LGR2    ' |
| NUMLX | 59 ints | 0 for non-LGR; LW*=0; LC*=2; LB*=1/2/3 |
| NUMLY | 59 ints | same pattern as NUMLX |
| NUMLZ | 59 ints | 0 for non-LGR; all LGR vectors = 1 |
| MEASRMNT | 236 chars | 4×59 — OPM gap, defer |
| UNITS | 59 chars | |
| STARTDAT | 6 ints | `{1, 1, 2015, 0, 0, 0}` |
| LGRNAMES | 2 chars | `{'LGR1    ', 'LGR2    '}` — sorted alphabetically |
| LGRVEC | 2 ints | `{21, 21}` = 2+19 per LGR |
| LGRTIMES | 2 ints | `{28, 23}` — different per LGR (LGR1 more ministeps than LGR2) |
| RUNTIMEI | 50 ints | |
| RUNTIMED | 5 doubles | |

## Key Findings

### LGRVEC=21 for both LGRs
19 LGR vectors per LGR (same structure as 1LGR) + 2 (TIME+YEARS) = 21.
Both LGRs have 19 vectors each — LGRVEC is symmetric in this test case.

### LGRTIMES asymmetry — CRITICAL
`{28, 23}` — LGR1 has more local ministeps than LGR2.
This proves LGRTIMES is per-LGR and reflects actual local timestepping history.
Cannot derive from `currentStep` alone for exact match.
For OPM approximation: use `currentStep` for all LGRs (acceptable until per-LGR step tracking is added).

### LGR vector interleaving in 2-LGR case
From idx 21 onwards, vectors alternate LGR1/LGR2:
  LWBHP/INJ(LGR1), LWBHP/PROD(LGR2), LWSTAT/INJ(LGR1), LWSTAT/PROD(LGR2), ...
This is NOT grouped by LGR. LGRS array correctly tracks which LGR each entry belongs to.
LGRVEC count-based derivation still works: count(entries == 'LGR1    ') = 19 → LGRVEC[0] = 21.

### LGRNAMES order
`{'LGR1    ', 'LGR2    '}` — sorted alphabetically. Matches the sorted-unique computation in PR02.

### NUMS for LC* in 2-LGR
LC* vectors have NUMS = -32767 (connection index not applicable at whole-well level).
This is consistent with 1LGR and global LC* handling.

### LB* cells in both LGRs
Each LGR has LBPR at 3 cells: (1,1,1), (2,2,1), (3,3,1) — same local coordinates in both LGRs.
NUMS for LB* = 1 (global linear index within that LGR's cell space).
LGRS field distinguishes which LGR each LBPR entry belongs to.

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR02 reference analysis | Created from Eclipse 2024.1 reference run of LGR-WELL-3X3-2LGR.DATA |
| 2026-05-05 | PR02 reference analysis | LGRTIMES={28,23} — asymmetric, per-LGR; currentStep approximation will differ |
| 2026-05-05 | PR02 reference analysis | LGR vectors interleaved (not grouped by LGR); LGRVEC still counts per LGR correctly |
| 2026-05-05 | PR02 reference analysis | LGRNAMES sorted alphabetically — matches PR02 sorted-unique LGRNAMES logic |
| 2026-05-05 | PR02 reference analysis | LGRVEC={21,21}; formula: 2 + count(params.lgrs == lgrname[i]) |
