# SHADOW: /home/arturcastiel/opm-hd/opm-tests/summary/LGR-WELL-3X3-1LGR.DATA
source-path: /home/arturcastiel/opm-hd/opm-tests/summary/LGR-WELL-3X3-1LGR.DATA
shadow-created: 2026-05-04
shadow-updated: 2026-05-04
git-hash: ec93abf2ce70aa8b615e2696c4f3b5b83bd1235fe83651f495e73120af99b5fc

## Summary
Test deck for LGR summary vector PR00 integration tests. 3x3x1 global grid, FIELD units. LGR1: CARFIN at global cell (1,1,1), 3x3x1 fine cells. INJ: gas injector via WELSPECL+COMPDATL in LGR1 fine cell (2,2,1). PROD: oil producer via WELSPECS+COMPDAT in global cell (3,3,1). SUMMARY requests: FOPR/FGOR/FWCT (field), WBHP/WOPR/WGIR/WWPR for both wells (well), BPR/BGSAT for 3 global cells (block), LWBHP/LWGIR/LWGIT/LWVIR/LWWIR/LWSTAT for INJ in LGR1 (LGR well), LCGFR/LCOFR/LCWFR/LCPR for INJ at (2,2,1) in LGR1 (LGR connection), LBPR/LBGSAT/LBSOIL/LBWSAT for LGR1 cells (LGR block). TSTEP: 12 monthly steps (Jan-Dec 2015).

## Key Structures
_(not yet analysed)_

## Dependencies
_(not yet analysed)_

## Architecture Role
_(not yet analysed)_

## Findings Log
| date | context | finding |
|------|---------|---------|

| 2026-05-04 | PR0 2026-05-04 | Used by test_Summary.cpp/LGR_Summary/lgr_config_1lgr. PR00 fix coverage: Fix1 (all 3 L second-letter branches exercised), Fix2 (lgr_ field populated for INJ, PROD has lgr_={} after PR01). PROD via WELSPECS/COMPDAT (not WELSPECL) confirms non-LGR wells are unaffected by LGR changes. |