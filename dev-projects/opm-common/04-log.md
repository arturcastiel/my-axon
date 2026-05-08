# Implementation Log — opm-common LGR Summary Vectors

---

## Entries

### [PR-001-harm] 2026-05-07 — Harmonization: rebase onto upstream/master + API migration

**Status:** API fixes applied, ready to build + test
**Branch:** lgr-summaryconfig (rebased onto upstream/master 20f7dcb15)

PR-000 was merged upstream (PR #5140). PR-001 depended on the old PR-000 API (`lgr_`/`.lgr(lgr_info{})`) and needed harmonization:

**Step 1 — Rebase:**
- `git rebase upstream/master` triggered conflict at commit 1/5 (old PR-000: 4ac7f0a5c)
- Resolved with `git rebase --skip` — old PR-000 commit superseded by upstream merge
- 4 PR-001 commits applied cleanly on top of new upstream/master

**Step 2 — API fixes in `SummaryConfig.cpp`:**
- `keywordLW`: dropped `lgr_info` local variable; `.lgr(lgr)` → `.lgr_name(lgr_name)`
- `keywordLB`: dropped `i`, `j`, `k` reads and `lgr_info`; `.lgr(lgr_info{...})` → `.lgr_name(lgr_name)`
- `keywordLC`: dropped `coords_defaulted`, `i`, `j`, `k`, and `lgr` local; → `.lgr_name(lgr_name)`

**Step 3 — Test updates in `test_SummaryConfigNode.cpp`:**
- `LGR_SummaryConfig_Deck` (6 tests): all `.lgr()` → `.lgr_name()`, all `->ijk[N]` dropped
- `LGR_Schema_Inline` (5 tests): same pattern; added BOOST_REQUIRE guards before dereference
- PR-001's 3-case `LGR` suite was already dropped during rebase (superseded by PR-000's 6-case suite)
- `ParseKeywords/Category` and `NoSumLgr` suites: no changes needed (no API dependency)

**Pending:** Step 10 — build and run `ctest -R "^(SummaryNode|SummaryConfigTests|Serialization)$"`

---

### [PR-000-rev1] 2026-05-07 — operator< simplification + expanded tests (review response)

**Status:** ready to commit
**Branch:** lgr-summary

Review feedback from @bska:
- `operator<` was needlessly complex manual logic; rewrite using `std::make_tuple` + `std::optional::operator<`
- Sort by LGR name *before* entity/number to group global vectors ahead of per-LGR vectors
- Add explicit unit tests for `operator==` and `operator<` for Block and Connection categories

Files changed:
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp` — `operator<` collapsed to 3 single-line `make_tuple` comparisons; LGR-first ordering in all categories
- `tests/test_SummaryConfigNode.cpp` — 3 new test cases: `lgr_operator_equal_block_and_connection`, `lgr_operator_less_lgr_before_entity`, `lgr_operator_less_null_before_set`

No drift from revised spec. `operator==` unchanged (already clean).

---

### [PR-000] 2026-05-04 — Data structures & category routing

**Status:** complete
**Branch:** lgr-summary (commit 4ac7f0a5c)

Files changed:
- `opm/io/eclipse/SummaryNode.cpp` — added `case 'L':` to `category_from_keyword()`
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.hpp` — added `lgr_name_` field, getter, setter; fixed `operator EclIO::SummaryNode()`; fixed `serializeOp()`
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp` — fixed `operator==` and `operator<` to include `lgr_name_` for Well/Block/Connection; `serializationTestObject()` covers `lgr_name_`
- `tests/test_SummaryNode.cpp` — `Category/LGR_category_routing` test case
- `tests/test_SummaryConfigNode.cpp` — 6 LGR test cases (3 original + 3 added per review)
- `opm/io/eclipse/SummaryNode.hpp` — **unchanged from master**

No drift from plan.

---

### [PR-001] 2026-05-05/06 — SummaryConfig routing + NOSUMLGR

**Status:** complete — pushed, Jenkins in progress
**Branch:** lgr-summaryconfig (4 commits, last: aaa3271e2)

Files changed:
- `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_WELL_PROBE` — new
- `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_CONNECTION_PROBE` — new
- `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_BLOCK_PROBE` — new
- `opm/input/eclipse/share/keywords/000_Eclipse100/N/NOSUMLGR` — new
- `opm/input/eclipse/share/keywords/keyword_list.cmake` — 4 entries added
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp` — `parseKeywordCategory case 'L':`, `keywordLW/LC/LB`, `handleKW dispatch`, `NOSUMLGR` in `is_processing_instruction`, `handleProcessingInstruction` branch
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.hpp` — `noSumLgr_` field, getter, `serializeOp`
- `tests/test_SummaryConfigNode.cpp` — 28 tests across 5 suites
- `tests/LGR-WELL-3X3-1LGR.DATA` — new test deck
- `tests/LGR-WELL-3X3-2LGR.DATA` — new test deck
- `CMakeLists_files.cmake` — 2 data file entries added

**Drift from plan:**
- `keyword_list.cmake` entries were initially missing → "Unknown keyword: LWBHP" on Jenkins. Added and squashed into commit 1.
- `LOCAL_WELL_PROBE` initially had `"size": 1` → parser rejected multi-record tables. Removed and squashed into commit 1.
- Added `LGR_Schema_Inline` suite (5 tests, not in original spec) — inline WELSPECL tests to catch schema bugs before Jenkins.

**Key lessons:**
- `keyword_list.cmake` must be updated alongside every new keyword JSON file
- `"size": 1` = single record; no `"size"` = table format. L* probes must use table format.
- Inline LW*/LC* tests require `WELSPECL` (not `WELSPECS`) to set `lgr_well_tag`

---

### [PR-002] 2026-05-06 — OutputStream SMSPEC LGR arrays

**Status:** committed, not pushed
**Branch:** lgr-smspec-pr2 (commit 128e785e6, may need stash pop)

Files changed:
- `opm/io/eclipse/OutputStream.cpp` — LGRS/NUMLX/Y/Z position fix, LGRVEC, LGRTIMES added
- `tests/test_OutputStream.cpp` — LGR_SingleGrid and LGR_TwoGrids test cases added

**Pending:** verify 3 fixes then push. See _meta.md "PR02 PENDING FIXES" for details.

---

### [PR-003] 2026-05-06 — LgrWellValue evaluator (LW* summary vectors)

**Status:** committed, pushed to origin (no PR yet)
**Branch:** lgr-lw-evaluators (commit on top of merge of lgr-summaryconfig + lgr-smspec-pr2)

Files changed:
- `opm/output/eclipse/Summary.cpp` — `lgr_well_units` map, `base_keyword()` helper, `LgrWellValue` class, `Factory::isLgrWellValue()` + `lgrWellValue()` methods, dispatch arm in `Factory::create()`
- `tests/test_Summary.cpp` — `LGR_well_factory_dispatch` and `LGR_well_wrong_lgr_produces_no_value` test cases; `lgr_well_deck` inline deck

**Build/test:** 45/45 tests pass

**Drift from plan:**
- Branch topology issue: `lgr-smspec-pr2` did not include PR01 keyword JSON files (LWOPR etc. were "unknown keyword"). Fix: created `lgr-lw-evaluators` from `lgr-summaryconfig`, then merged `lgr-smspec-pr2` in.
- WOPR/WBHP inline deck format bug: initially had trailing `/` per well name (table format) instead of correct WELL_PROBE format (well name only, single `/` block terminator). Fixed before commit.
- `Summary(SummaryConfig&, ...)` takes non-const ref — test variables must be `auto config` not `const auto config`. Already fixed in previous session.

**Key lessons:**
- WELL_PROBE (WOPR, WBHP): no trailing `/` per well name. Single `/` closes the block.
- LOCAL_WELL_PROBE (LWOPR, LWBHP): table format, each record ends with `/`, block ends with `/`
- LW* keywords require the `lgr-summaryconfig` branch in history — keyword JSONs only exist there
