# PR List — opm-common LGR Summary Vectors

Generated: 2026-05-06
Source: claude/lgr-summary-vectors/LGR_Implementation_Masterplan.md

---

## PR-000 — Data structures: lgr_ field in SummaryConfigNode + category routing

**Complexity:** small
**Repo:** opm-common
**Branch:** lgr-summary
**Status:** ✅ pushed (commit 4ac7f0a5c)
**Depends on:** nothing
**Scope:** SummaryNode.hpp, SummaryNode.cpp, SummaryConfig.hpp, SummaryConfig.cpp

---

## PR-001 — SummaryConfig: parse LW*/LC*/LB* keywords + NOSUMLGR

**Complexity:** large
**Repo:** opm-common
**Branch:** lgr-summaryconfig
**Status:** ✅ pushed (commit aaa3271e2) — Jenkins pending
**Depends on:** PR-000
**Scope:** keyword JSON files (4), SummaryConfig.cpp, SummaryConfig.hpp, test_SummaryConfigNode.cpp, test decks

---

## PR-002 — OutputStream: SMSPEC LGR metadata arrays

**Complexity:** medium
**Repo:** opm-common
**Branch:** lgr-smspec-pr2
**Status:** ✅ committed, pushed
**Depends on:** PR-000
**Scope:** OutputStream.cpp, OutputStream.hpp, test_OutputStream.cpp

---

## PR-003 — Summary: LW* well evaluators

**Complexity:** large
**Repo:** opm-common
**Branch:** lgr-lw-evaluators
**Status:** ✅ committed, pushed — GitHub PR pending
**Depends on:** PR-001, PR-002
**Scope:** opm/output/eclipse/Summary.cpp — new LgrWellValue evaluator

---

## PR-004 — Summary: LC* connection evaluators

**Complexity:** large
**Repo:** opm-common
**Branch:** (not started)
**Status:** □ not started
**Depends on:** PR-001, PR-002
**Scope:** opm/output/eclipse/Summary.cpp — new LgrConnectionValue evaluator

---

## PR-005 — Integration tests LW* + LC*

**Complexity:** medium
**Repo:** opm-common
**Branch:** (not started)
**Status:** □ not started
**Depends on:** PR-003, PR-004, PR_TESTDECK
**Scope:** tests/

---

## PR-006 — Simulator: LB* block evaluators

**Complexity:** large
**Repo:** opm-common + opm-simulators
**Branch:** (not started)
**Status:** □ not started
**Depends on:** PR-003, PR-004
**Scope:** opm/output/eclipse/Summary.cpp, opm-simulators GenericOutputBlackoilModule
