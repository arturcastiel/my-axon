# Plan — opm-common LGR Summary Vectors
phase:    2-plan
updated:  2026-05-06
status:   complete — PRs 0-2 specified and partially implemented

---

## High-Level Plan

Add Eclipse-compatible LGR summary vector support across three layers:
1. **Config layer** — parse LW*/LC*/LB* deck keywords into typed `SummaryConfigNode` objects with LGR identity
2. **IO layer** — write correct SMSPEC LGR metadata arrays (LGRS, NUMLX/Y/Z, LGRNAMES)
3. **Evaluation layer** — compute LW*/LC*/LB* values at each timestep and write to UNSMRY

Each layer is a separate PR (or two). No PR crosses layer boundaries.

### Key architectural constraints
- No PR depends on a later one — strict linear dependency order
- `SummaryConfigNode` changes (PR0) are the foundation for all others
- SMSPEC write (PR2) can be developed in parallel with config parse (PR1) since both only depend on PR0
- Evaluators (PR3/PR4) depend on both PR1 (nodes exist) and PR2 (SMSPEC correctly written)
- LB* evaluators (PR6) need simulator-layer data — goes in opm-simulators

### Split rationale
- PR0 is tiny (4 files, ~90 lines) and has zero risk — merge it first
- PR1 is large but pure parsing — no output, completely safe to merge before evaluators exist
- PR2 is medium, independent of PR1 — can proceed in parallel
- PR3/PR4 are the risky ones — real values written; gate on PR1+PR2 both merged

---

## PR Dependency Graph

```
PR0  ──►  PR1  ──►  PR3  ──►  PR5
     │         │              ▲
     │         └──►  PR4  ───┘
     │                        │
     └──►  PR2  ──────────────┘
                               │
           PR_TESTDECK ────────┤
                               │
                               └──►  PR6  (opm-simulators)
```

---

## PR List

| PR | Title | Layer | Status | Branch |
|----|-------|-------|--------|--------|
| PR-000 | Data structures: lgr_ field + category routing | foundation | ✅ pushed | lgr-summary |
| PR-001 | SummaryConfig: parse LW*/LC*/LB* + NOSUMLGR | config | ✅ pushed | lgr-summaryconfig |
| PR-002 | OutputStream: SMSPEC LGR metadata arrays | IO | ⚠ committed | lgr-smspec-pr2 |
| PR-003 | Summary: LW* well evaluators | evaluation | □ not started | — |
| PR-004 | Summary: LC* connection evaluators | evaluation | □ not started | — |
| PR-005 | Integration tests LW* + LC* | testing | □ not started | — |
| PR-006 | Simulator: LB* block evaluators | evaluation (simulators) | □ not started | — |

---

## Files Affected by Each Layer

### Foundation (PR0)
- `opm/io/eclipse/SummaryNode.hpp` — lgr_info struct extensions
- `opm/io/eclipse/SummaryNode.cpp` — category_from_keyword case 'L'
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.hpp` — lgr_ field
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp` — operator== / operator<

### Config layer (PR1)
- `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_*_PROBE` — new
- `opm/input/eclipse/share/keywords/000_Eclipse100/N/NOSUMLGR` — new
- `opm/input/eclipse/share/keywords/keyword_list.cmake` — 4 entries
- `SummaryConfig.cpp` — parseKeywordCategory, keywordLW/LC/LB, handleKW dispatch, NOSUMLGR
- `SummaryConfig.hpp` — noSumLgr_ field

### IO layer (PR2)
- `opm/io/eclipse/OutputStream.hpp` — Parameters struct extensions
- `opm/io/eclipse/OutputStream.cpp` — LGRS/NUMLX/Y/Z/LGRNAMES/LGRVEC/LGRTIMES in write()

### Evaluation layer (PR3/PR4)
- `opm/output/eclipse/Summary.cpp` — new evaluator classes + factory dispatch

### Evaluation layer — block (PR6)
- `opm/output/eclipse/Summary.cpp` (opm-common) — LgrBlockValue evaluator
- `opm-simulators/opm/simulators/wells/BlackoilWellModelGeneric.cpp` (or similar) — block data collection

---

## Next Steps

1. Restore stash on `lgr-smspec-pr2`, verify 3 fixes, push PR-002
2. Run Study 5 (read `Summary.cpp` lines 4650-4800) before writing PR-003.md
3. Write PR-003.md using `code-dev pr 3`
4. Implement PR-003, log with `code-dev log`
