# SHADOW: /home/arturcastiel/opm-hd/opm-tests/summary/LGR-WELL-3X3-2LGR.DATA
source-path: /home/arturcastiel/opm-hd/opm-tests/summary/LGR-WELL-3X3-2LGR.DATA
shadow-created: 2026-05-04
shadow-updated: 2026-05-04
git-hash: 0330a101edc0fb06bde1c9c2111583ffc9bff050205f486adb8eb1f0ad15d477

## Summary
Test deck for LGR summary vector PR00 Fix4 dedup stress test. 3x3x1 global grid, FIELD units. LGR1: CARFIN at (1,1,1), 3x3x1 — INJ here. LGR2: CARFIN at (3,3,1), 3x3x1 — PROD here. Both wells via WELSPECL+COMPDATL. SUMMARY requests: same W*/B*/F* as 1LGR deck, plus LWBHP/LWSTAT for both LGRs (primary Fix4 Well test), LCPR/LCGFR/LCOFR/LCWFR for both wells (Fix4 Connection), LBPR/LBGSAT for both LGRs x 3 cells each = 6 entries, LBSOIL/LBWSAT for both LGRs at (2,2,1) (Fix4 Block direct test). TSTEP: 12 monthly steps.

## Key Structures
_(not yet analysed)_

## Dependencies
_(not yet analysed)_

## Architecture Role
_(not yet analysed)_

## Findings Log
| date | context | finding |
|------|---------|---------|

| 2026-05-04 | PR0 2026-05-04 | Used by test_Summary.cpp/LGR_Summary/lgr_config_2lgr. PR00 Fix4 stress test: LBPR has 'LGR1' 2 2 1 AND 'LGR2' 2 2 1 — same keyword+ijk, different LGR name. Without lgr_ in Block operator== one node is silently dropped. Fix4 deck-level assertion (expect 6 LBPR nodes) deferred to PR01 TODO in test; operator== fix already covered by unit test lgr_dedup_distinct_lgrs in test_SummaryConfigNode.cpp. |