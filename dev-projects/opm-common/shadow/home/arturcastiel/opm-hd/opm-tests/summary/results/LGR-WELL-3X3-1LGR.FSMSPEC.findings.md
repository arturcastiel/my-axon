# Shadow: opm-tests/summary/results/LGR-WELL-3X3-1LGR.FSMSPEC
project: opm-common
source: Eclipse reference simulator output (ASCII formatted)
indexed: 2026-05-05

## Summary
Eclipse reference SMSPEC for the 1-LGR test case. 40 total vectors:
21 non-LGR (TIME, YEARS, F*, W*, BPR, BGSAT) + 19 LGR (LW*, LC*, LB* for INJ in LGR1).
Ground truth for OPM's SMSPEC output format.

## Array Inventory
| Array | Count | Notes |
|---|---|---|
| INTEHEAD | 2 ints | `{2, 100}` — FIELD units, ECLIPSE100 |
| RESTART | 9 chars | all blank (fresh run) |
| DIMENS | 6 ints | `{40, 3, 3, 1, 0, -1}` |
| KEYWORDS | 40 chars | |
| WGNAMES | 40 chars | |
| NUMS | 40 ints | |
| LGRS | 40 chars | indices 0–20 blank; indices 21–39 = 'LGR1    ' |
| NUMLX | 40 ints | 0 for non-LGR; LW*=0, LC*=2, LB*=1/2/3 |
| NUMLY | 40 ints | 0 for non-LGR; LW*=0, LC*=2, LB*=1/2/3 |
| NUMLZ | 40 ints | 0 for non-LGR; LW*=0, LC*=1, LB*=1 |
| MEASRMNT | 160 chars | 4×40 — measurement type descriptions (OPM does not emit this) |
| UNITS | 40 chars | |
| STARTDAT | 6 ints | `{1, 1, 2015, 0, 0, 0}` |
| LGRNAMES | 1 char | `{'LGR1    '}` |
| LGRVEC | 1 int | `{21}` = 2 (TIME+YEARS in LGR file) + 19 (LGR vectors) |
| LGRTIMES | 1 int | `{23}` = total local ministeps written |
| RUNTIMEI | 50 ints | simulation metadata |
| RUNTIMED | 5 doubles | `{365.0, 2.066, 2.066, 0, 0}` |

## Key Findings

### LGRS/NUMLX/NUMLY/NUMLZ position
Appear **after NUMS, before MEASRMNT/UNITS** — NOT after STARTDAT.
PR02 committed (128e785e6) has these in the wrong position (after STARTDAT).

### LGRVEC semantics
Value = 21 = 2 + 19.
- 2: TIME and YEARS that appear in every LGR's section in ROOT.LGR file
- 19: the 19 LGR vectors assigned to LGR1 (LWBHP, LWGIR, ... LBWSAT)
Formula: `2 + count(params.lgrs entries matching this LGR name)`

### LGRTIMES semantics
Value = 23 (12 TSTEP report steps + sub-steps / initial step).
This is dynamic — changes as simulation advances.
For OPM implementation: use `currentStep` as the value for all LGRs.

### MEASRMNT
4×N char array (160 chars = 4×40). Per-vector 32-char measurement description.
Format: `"{letter}:{description}"` padded to 32 chars in 4×8-char chunks.
Examples: `"O:Simulation_Time"`, `"O:Liquid_Flowrate"`, `"O:Pressure"`.
OPM does NOT emit this. Pre-existing gap — not LGR-specific. Defer to separate PR.

### RUNTIMED
5 doubles: `{sim_duration_days, cpu_time, cpu_time, 0, 0}`.
OPM does NOT emit this. Pre-existing gap. Defer to separate PR.

### LW* NUMLX/NUMLY/NUMLZ = 0
Whole-well LGR vectors (LW*) have local ijk = 0/0/0 in NUMLX/NUMLY/NUMLZ.
This matches the `lgr_info::ijk = {0,0,0}` sentinel in SummaryNode.hpp.

### LC* NUMLX/NUMLY/NUMLZ
Connection vectors have actual local cell coordinates.
1LGR: LCGFR/LCOFR/LCWFR/LCPR all at (2,2,1) — INJ perforation in LGR1.

### LB* NUMLX/NUMLY/NUMLZ
Block vectors have actual local cell I/J/K.
1LGR: LBPR at I=1,2,3 / J=1,2,3 / K=1 (3 cells in LGR1).

## Vector Breakdown (1LGR, 40 total)
```
idx  0  TIME         non-LGR
idx  1  YEARS        non-LGR
idx  2  FOPR/FIELD   non-LGR
idx  3  FGOR/FIELD   non-LGR
idx  4  FWCT/FIELD   non-LGR
idx  5  FWIR/FIELD   non-LGR
idx  6  WBHP/INJ     non-LGR
idx  7  WBHP/PROD    non-LGR
idx  8  WOPR/INJ     non-LGR
idx  9  WOPR/PROD    non-LGR
idx 10  WGIR/INJ     non-LGR
idx 11  WGIR/PROD    non-LGR
idx 12  WGOR/PROD    non-LGR
idx 13  WWPR/INJ     non-LGR
idx 14  WWPR/PROD    non-LGR
idx 15  BPR(1)       non-LGR
idx 16  BPR(5)       non-LGR
idx 17  BPR(9)       non-LGR
idx 18  BGSAT(1)     non-LGR
idx 19  BGSAT(5)     non-LGR
idx 20  BGSAT(9)     non-LGR   ← 21 non-LGR vectors
idx 21  LWBHP/INJ    LGR1 (0,0,0)
idx 22  LWGIR/INJ    LGR1 (0,0,0)
idx 23  LWGIT/INJ    LGR1 (0,0,0)
idx 24  LWVIR/INJ    LGR1 (0,0,0)
idx 25  LWVIT/INJ    LGR1 (0,0,0)
idx 26  LWWIR/INJ    LGR1 (0,0,0)
idx 27  LWSTAT/INJ   LGR1 (0,0,0)
idx 28  LCGFR/INJ    LGR1 (2,2,1)
idx 29  LCOFR/INJ    LGR1 (2,2,1)
idx 30  LCWFR/INJ    LGR1 (2,2,1)
idx 31  LCPR/INJ     LGR1 (2,2,1)
idx 32  LBPR(1)      LGR1 (1,1,1)
idx 33  LBPR(1)      LGR1 (2,2,1)
idx 34  LBPR(1)      LGR1 (3,3,1)
idx 35  LBGSAT(1)    LGR1 (1,1,1)
idx 36  LBGSAT(1)    LGR1 (2,2,1)
idx 37  LBGSAT(1)    LGR1 (3,3,1)
idx 38  LBSOIL       LGR1 (2,2,1)
idx 39  LBWSAT       LGR1 (2,2,1)   ← 19 LGR vectors → LGRVEC = 2+19 = 21
```

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR02 reference analysis | Created from Eclipse 2024.1 reference run of LGR-WELL-3X3-1LGR.DATA |
| 2026-05-05 | PR02 reference analysis | LGRS/NUMLX/NUMLY/NUMLZ position: after NUMS, before MEASRMNT/UNITS (not after STARTDAT) |
| 2026-05-05 | PR02 reference analysis | LGRVEC=21 = 2 + 19 (TIME+YEARS in LGR file + 19 LGR vectors) |
| 2026-05-05 | PR02 reference analysis | LGRTIMES=23 = local ministeps written; approximated by currentStep in OPM |
| 2026-05-05 | PR02 reference analysis | MEASRMNT and RUNTIMED present in reference; OPM gap; defer to separate PR |
