# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/tests/test_Summary.cpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/tests/test_Summary.cpp
shadow-created: 2026-05-04
shadow-updated: 2026-05-04
git-hash: 077cc17198a94c3f11533c60038eb3eb67247bf0431ac88ea9e2b6a77864a6c9

## Summary
Integration test for Summary output (opm/output/eclipse/Summary.cpp). Uses 'setup' struct: parses deck, builds EclipseState+Schedule+SummaryConfig, creates synthetic well data, writes SMSPEC+UNSMRY, reads back via ESmry. WorkArea ta creates a temp directory. LGR_Summary suite runs from CWD=/home/arturcastiel/opm-hd/opm-tests/summary (set in opm.code-workspace launch.json). Test binary: build/opm-common/bin/test_Summary.

## Key Structures
_(not yet analysed)_

## Dependencies
_(not yet analysed)_

## Architecture Role
_(not yet analysed)_

## Findings Log
| date | context | finding |
|------|---------|---------|

| 2026-05-04 | PR0 2026-05-04 | PR0 applied 2026-05-04: replaced KEYWORD1 placeholder with lgr_config_1lgr (LGR-WELL-3X3-1LGR.DATA) and lgr_config_2lgr (LGR-WELL-3X3-2LGR.DATA). lgr_config_1lgr: asserts no-throw parse with WELSPECL/COMPDATL/CARFIN, WBHP/WOPR/BPR node counts (2/2/3), LGR keyword presence as Misc (LWBHP/LCGFR/LBPR), and category_from_keyword routing for all 3 L second-letter branches. lgr_config_2lgr: same for 2LGR deck with both wells in LGRs; includes TODO for PR01 Fix4 LBPR 6-node dedup assertion. |