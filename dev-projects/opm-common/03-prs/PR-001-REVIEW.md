# PR-001 Code Review — LGR Summary Vector Support in SummaryConfig

**Branch:** `lgr-summaryconfig`  
**Base:** `upstream/master` (commit `20f7dcb15`)  
**Commits:** 3  
**Files touched:** 11  
**Reviewer:** internal  
**Date:** 2026-05-07

---

## Commits under review

```
56cc3cb89  keyword: add parser schema for LGR summary vectors and NOSUMLGR
dbfeddd15  SummaryConfig: parse LW*/LC*/LB* keywords and handle NOSUMLGR
f3f0c7a63  tests: SummaryConfig LGR keyword parsing and NOSUMLGR
```

---

## Section 1 — Keyword JSON Schemas

### Files
- `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_WELL_PROBE`
- `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_CONNECTION_PROBE`
- `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_BLOCK_PROBE`
- `opm/input/eclipse/share/keywords/000_Eclipse100/N/NOSUMLGR`
- `opm/input/eclipse/share/keywords/keyword_list.cmake`

### Findings

**NOTE — Schema: `LOCAL_WELL_PROBE`**  
The absence of `"size"` is intentional (table format, multiple records). The test `lw_multi_record_produces_two_nodes` explicitly guards against a `"size":1` regression. The `deck_name_regex: "LW.+"` is correct and the test `lw_regex_matches_lwstat` validates it. The `"size_type": "ALL"` on the WELLS item is consistent with WELL_PROBE and allows wildcard well patterns.

**NOTE — Schema: `LOCAL_CONNECTION_PROBE`**  
The comment "Note: LCOFRU/LCWFRU/LCGFRU are handled by CONNECTION_PROBE" documents a known scope exclusion: these three keywords are registered under CONNECTION_PROBE's explicit `deck_names` and have a different record format (WELL, I, J, K — no LGR_NAME). The comment is correct but see BLOCKER-1 below regarding the runtime dispatch path.

**NOTE — Schema: `LOCAL_BLOCK_PROBE`**  
The comment "Note: LBPR_X/LBSCN_X/LBHD_X/LBHDF_X/LBCTRA_X are handled by BLOCK_PROBE" similarly documents tracer-variant keywords that remain in BLOCK_PROBE. Same routing concern as above (BLOCKER-1).

**NOTE — Schema: `NOSUMLGR`**  
Minimal schema (name + sections only). This is correct — the keyword has no items and no records. Consistent with how other processing instruction keywords are defined.

**NOTE — `keyword_list.cmake` ordering**  
Both L-group entries (LOCAL_BLOCK_PROBE, LOCAL_CONNECTION_PROBE, LOCAL_WELL_PROBE) are inserted in alphabetical order within the L block between LKRWR and LOAD. NOSUMLGR is inserted between NOSIM and NOWARN in the N block. Both placements are correct.

---

## Section 2 — `SummaryConfig.cpp`

### File
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp`

### Findings

**BLOCKER-1 — Wrong dispatch for LCOFRU, LCWFRU, LCGFRU, LBPR_X, LBSCN_X, etc.**  
Location: `SummaryConfig.cpp:1800-1805` (Connection dispatch), `1789-1793` (Block dispatch)

The dispatch switch routes all `Category::Connection` keywords where `keyword.name()[0] == 'L'` to `keywordLC`, and all `Category::Block` keywords where `keyword.name()[0] == 'L'` to `keywordLB`. This is too broad.

LCOFRU, LCWFRU, LCGFRU are in CONNECTION_PROBE's `deck_names` (not LOCAL_CONNECTION_PROBE). Their record format is (WELL, I, J, K) with no LGR_NAME. When routed to `keywordLC`, item(0) is read as LGR_NAME (getting the well name string instead), item(1) is read as well_pattern (getting the I integer cast to string). The `schedule.wellNames(I_as_string)` call returns empty, `handleMissingWell` fires, and zero nodes are produced — silently.

The same issue affects LBPR_X, LBSCN_X, LBHDF_X, LBHD_X, LBCTRA_X under BLOCK_PROBE: their record format is (I, J, K) with no LGR_NAME. Routed to `keywordLB`, item(0) is read as LGR_NAME, receiving an integer value coerced to string. These nodes are nonsensical.

The schema comments document the exclusion correctly but the dispatch code does not enforce it.

Recommended fix: make the dispatch check match on the schema name rather than the first character. The cleanest approach is to check whether the keyword has LGR_NAME as its first item at construction time, or maintain an explicit exclusion set:

```cpp
// In the Connection case:
if (keyword.name()[0] == 'L'
    && !is_in_set({"LCOFRU", "LCWFRU", "LCGFRU"}, keyword.name()))
{
    keywordLC(...);
} else {
    connectionKeyword(...);
}
```

Similarly for Block. An alternative is to add these keywords to `LOCAL_CONNECTION_PROBE` / `LOCAL_BLOCK_PROBE` as exclusions in their regexes, but explicit set membership is more readable.

This bug is latent — it only fires when LCOFRU/LCWFRU/LCGFRU or LBPR_X appear in a SUMMARY deck, which may be uncommon in the current test suite. But it is a correctness defect.

---

**BLOCKER-2 — `SummaryConfig::operator==` does not include `noSumLgr_`**  
Location: `SummaryConfig.cpp:2503-2509`

```cpp
bool SummaryConfig::operator==(const Opm::SummaryConfig& data) const
{
    return (this->m_keywords == data.m_keywords)
        && (this->short_keywords == data.short_keywords)
        && (this->summary_keywords == data.summary_keywords)
        ;
}
```

`noSumLgr_` is included in `serializeOp` (SummaryConfig.hpp:581) but omitted from `operator==`. This means two `SummaryConfig` objects that differ only in the NOSUMLGR flag compare as equal. The round-trip via serialization preserves the flag, but equality comparison does not reflect it.

Note: `runSummaryConfig` (RUNSUM/NARROW/SEPARATE) is excluded from both `serializeOp` and `operator==` — that is a deliberate pre-existing pattern because those flags are processing-time ephemera. `noSumLgr_`, however, IS included in `serializeOp`, which means the PR author considers it part of the persistent state. Omitting it from `operator==` is therefore an oversight.

Recommended fix:
```cpp
bool SummaryConfig::operator==(const Opm::SummaryConfig& data) const
{
    return (this->m_keywords     == data.m_keywords)
        && (this->short_keywords  == data.short_keywords)
        && (this->summary_keywords == data.summary_keywords)
        && (this->noSumLgr_       == data.noSumLgr_)
        ;
}
```

---

**BLOCKER-3 — `keywordLB` and `keywordLC` do not read I,J,K; multi-record dedup collapses distinct cells**  
Location: `SummaryConfig.cpp:1101-1105` (keywordLB), `1457-1476` (keywordLC)

`keywordLB` reads only item(0) (LGR_NAME) and ignores items(1,2,3) (I,J,K). All LB* records for the same LGR within one keyword produce nodes with identical fields (`keyword`, `category`, `lgr_name`, `number_=INT_MIN`). After `uniq()`, they deduplicate to a single node regardless of how many distinct cells were requested.

`keywordLC` similarly ignores items(2,3,4) (I,J,K). Two LC* records for the same well in the same LGR at different cells deduplicate to one node.

The context note says "ijk coordinates are an IO-layer concern (PR-002 fills them from SMSPEC arrays)." If this is intentional, the tests must not assert exact counts for multi-record same-LGR keywords. But the test `lc_multi_record_produces_nodes` (test_SummaryConfigNode.cpp:642-658) asserts `nodes.size() == 2` for two LCOFR records with the same well and same LGR at different I,J,K. This assertion **will fail** after dedup.

Similarly `lb_multi_record_produces_nodes` (test_SummaryConfigNode.cpp:660-675) asserts `nodes.size() == 2` for two LBPR records for the same LGR at different cells. This assertion **will fail**.

There are two ways to resolve this:

Option A (preferred if IJK deferral is intentional): read I,J,K in `keywordLB` and `keywordLC` and store them in `number_` as a locally-linear index (consistent with how `keywordB` stores global linear indices). This allows correct dedup and gives PR-002 something to work with. The test assertions become valid.

Option B (if IJK remains fully deferred to PR-002): change the test assertions to `BOOST_CHECK_EQUAL(nodes.size(), 1u)` for same-LGR multi-record cases, and document that multiple cells in the same LGR are currently collapsed.

Option A is strongly preferred because losing distinct cell tracking at the config layer means the SMSPEC file cannot be correctly reconstructed.

---

**SUGGESTION-1 — `keywordLW` well-filter comments are missing**  
Location: `SummaryConfig.cpp:864-896`

`keywordLW` filters wells by `get_lgr_well_tag() != lgr_name` (line 891-893). This is the correct mechanism for LGR well discrimination but is not documented. A short comment explaining why the filter is needed (wells declared with WELSPECL carry an LGR tag; WELSPECS wells have an empty tag) would help reviewers.

---

**SUGGESTION-2 — `handleMissingWell` is called before knowing any well passes the LGR filter**  
Location: `SummaryConfig.cpp:885-887` (keywordLW), `1466-1468` (keywordLC)

If `schedule.wellNames(pattern)` returns results but none pass the LGR filter, `handleMissingWell` was not called at all — a silent empty result. If `schedule.wellNames(pattern)` returns empty, `handleMissingWell` fires for "no such well" which may be misleading when the well exists but lives in a different LGR. Consider checking candidates-after-filter-empty separately and logging a more targeted diagnostic:
```
"Well pattern {} matched no wells in LGR {}"
```
This is a usability improvement, not a correctness defect.

---

**NOTE — `parseKeywordType` returns `Undefined` for all LW*/LC*/LB* keywords**  
Location: `SummaryConfig.cpp:1928-1951`

`is_pressure("LWBHP")` checks `"LWBHP".substr(1) == "WBHP"` against the pressure set (which contains "BHP" not "WBHP") → false. All LGR keywords return `Type::Undefined`. This is a pre-existing gap in the type classifier that the PR does not address. The PR's own use of `parseKeywordType` for LGR nodes is consistent with the rest of the codebase (all keyword handlers call it). A follow-on fix to `is_pressure` / `is_rate` / `is_total` to strip one additional prefix level for L-keywords would be welcome but is not a blocker for this PR's scope.

---

## Section 3 — `SummaryConfig.hpp`

### File
- `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.hpp`

### Findings

**SUGGESTION-3 — `handleProcessingInstruction` doc comment is stale**  
Location: `SummaryConfig.hpp:682-685`

```cpp
/// \param[in] keyword Run summary directive.  Expected to be one of
/// "RUNSUM", "NARROW", or "SEPARATE" which affect the 'create',
/// 'narrow', and 'separate' members of \c runSummaryConfig
/// respectively.
```

NOSUMLGR is not mentioned. Recommended fix: append "or 'NOSUMLGR' which sets `noSumLgr_`" to the list.

---

**NOTE — `noSumLgr_` comment uses forward reference to "Phase 2"**  
Location: `SummaryConfig.hpp:675`

```cpp
/// Whether the NOSUMLGR keyword was present in the deck.
/// When true, the ROOT.LGR file should be suppressed (Phase 2).
bool noSumLgr_ { false };
```

"Phase 2" is a private project label with no meaning to an external reviewer. Preferred phrasing: "When true, the simulator should suppress the ROOT.LGR file (handled by the output layer)."

---

**NOTE — `lgr_info{}` construction in conversion operator**  
Location: `SummaryConfig.hpp:165-167`

```cpp
auto lgr = this->lgr_name_.has_value()
    ? std::optional<EclIO::lgr_info>{ EclIO::lgr_info{ *this->lgr_name_, {} } }
    : std::optional<EclIO::lgr_info>{};
```

The `{}` (zero-initialized ijk array) is an explicit placeholder indicating PR-002 will fill in the real coordinates. This is correct by design. A brief inline comment like `/* ijk filled by IO layer */` would make the intent clear.

---

## Section 4 — `tests/test_SummaryConfigNode.cpp`

### Test suites added by this PR
- `ParseKeywords/Category` — 5 cases
- `LGR` — 6 cases (owned partly by PR-000)
- `NoSumLgr` — 2 cases
- `LGR_SummaryConfig_Deck` — 6 cases
- `LGR_Schema_Inline` — 5 cases

### Findings

**BLOCKER-4 — `lc_multi_record_produces_nodes` asserts wrong count**  
Location: `test_SummaryConfigNode.cpp:642-658`

```cpp
BOOST_CHECK_EQUAL(nodes.size(), 2u);
```

The test constructs two LCOFR records for the same well (`WELL1`) in the same LGR (`LGR1`) at different I,J,K coordinates. Because `keywordLC` does not read I,J,K, both records produce nodes with identical `(keyword, namedEntity, number=INT_MIN, lgr_name)`. After `uniq()`, they deduplicate to 1. The assertion of 2 nodes is incorrect and will fail at runtime.

This is the test-side manifestation of BLOCKER-3. Fix by either implementing Option A from BLOCKER-3 (read IJK into `number_`), or changing the assertion to 1 with documentation of the limitation.

---

**BLOCKER-5 — `lb_multi_record_produces_nodes` asserts wrong count**  
Location: `test_SummaryConfigNode.cpp:660-675`

```cpp
BOOST_REQUIRE_EQUAL(nodes.size(), 2u);
```

Same issue as BLOCKER-4 but for LBPR. Two records `'LGR1' 1 1 1` and `'LGR1' 2 2 1` produce identical nodes after `keywordLB` ignores I,J,K. Deduplicated to 1 by `uniq()`. Assertion fails.

---

**NOTE — Missing deck-level NOSUMLGR test**  
Location: `test_SummaryConfigNode.cpp:396-412`

The `NoSumLgr` suite has two tests: `default_is_false` and `serialization_test_object_sets_flag`. There is no test that parses a deck containing the NOSUMLGR keyword and verifies `cfg.noSumLgr() == true`. This would be a three-line addition to the suite:

```cpp
BOOST_AUTO_TEST_CASE(deck_with_nosumlgr_sets_flag)
{
    const auto cfg = parseSummarySection("NOSUMLGR\n");
    BOOST_CHECK(cfg.noSumLgr());
}
```

(Using the `parseSummarySection` helper from `LGR_Schema_Inline`.) This gap means the actual `handleProcessingInstruction("NOSUMLGR")` path is not exercised by any test.

---

**NOTE — `lgr_1lgr_lb_node_populated` test masks the dedup problem**  
Location: `test_SummaryConfigNode.cpp:463-478`

The test uses `BOOST_CHECK(!hits.empty())` (at-least-one) rather than `BOOST_CHECK_EQUAL(hits.size(), 3u)` for a 1LGR deck that requests LBPR for three cells. This was either a deliberate hedge or a sign the author anticipated the dedup. The hedge is acceptable for a deck-level smoke test, but it obscures the collapse of three records to one.

---

**NOTE — `lw_regex_matches_lwstat` is a good regression test**  
Location: `test_SummaryConfigNode.cpp:677-690`

Positive note: the test correctly uses the inline-schema helper with 'INJ' declared in LGR1 via WELSPECL, confirming the regex `LW.+` matches LWSTAT and the LGR filter works end-to-end. No issues.

---

**NOTE — `parseSummarySection` SCHEDULE stub has no COMPDAT**  
Location: `test_SummaryConfigNode.cpp:555-601`

The stub schedule has WELSPECL but no COMPDATL/COMPDAT. This means wells have no connections. For `keywordLC`, the test verifies that nodes are produced by matching `get_lgr_well_tag()` — this works at the well level without connection data. However, if connection-level semantics (e.g., actual connection count validation) are added to `keywordLC` in a future PR, these tests will need COMPDATL entries. Note this in a future work comment if desired.

---

## Section 5 — Test Data Files

### Files
- `tests/LGR-WELL-3X3-1LGR.DATA`
- `tests/LGR-WELL-3X3-2LGR.DATA`

### Findings

**NOTE — Grid layout and well placement are well-documented**  
Both files include ASCII grid diagrams in their headers, PR coverage notes (Fix 1/2/3/4), and per-keyword comments. The 2LGR file correctly isolates Fix 4 (dedup across two LGRs with same-ijk block keywords). This is good practice.

**NOTE — 1LGR deck has no NOSUMLGR keyword**  
Neither deck exercises the NOSUMLGR path. This reinforces the gap noted in NOTE above (Missing deck-level NOSUMLGR test). A variant of one of these decks with NOSUMLGR would be useful, though the inline `parseSummarySection` helper is a lighter alternative.

**NOTE — `LBPR` in 1LGR deck requests 3 records but only 1 node survives dedup**  
```
LBPR
  'LGR1'  1  1  1 /
  'LGR1'  2  2  1 /
  'LGR1'  3  3  1 /
/
```
Three records → one node (all deduplicate due to BLOCKER-3). The deck comment says "Three cells: corners (1,1,1), (3,3,1) and centre (2,2,1)" but the actual SummaryConfig produced has only one Block node for LGR1. The deck is correct as written; the comment does not claim three nodes will result. However, this is misleading once BLOCKER-3 is understood.

---

## Section 6 — `CMakeLists_files.cmake`

### Findings

**NOTE — Placement correct**  
`tests/LGR-WELL-3X3-1LGR.DATA` and `tests/LGR-WELL-3X3-2LGR.DATA` are added in alphabetical order within the `tests/LGR*` block (lines 724-725). `tests/test_SummaryConfigNode.cpp` is added at line 560, alphabetically correct between `test_SummaryNode.cpp` and `test_Summary_Group.cpp`. No issues.

---

## Summary of Findings

| ID | Severity | File | Description |
|----|----------|------|-------------|
| BLOCKER-1 | BLOCKER | SummaryConfig.cpp:1789-1805 | LCOFRU/LCWFRU/LCGFRU and LBPR_X/LBSCN_X/etc. are routed to keywordLC/keywordLB despite having CONNECTION_PROBE/BLOCK_PROBE record formats; wrong items are read |
| BLOCKER-2 | BLOCKER | SummaryConfig.cpp:2503-2509 | `SummaryConfig::operator==` omits `noSumLgr_` despite it being in `serializeOp`; equality comparison is inconsistent with serialization |
| BLOCKER-3 | BLOCKER | SummaryConfig.cpp:1101-1105, 1457-1476 | `keywordLB` and `keywordLC` ignore I,J,K fields; multiple records for the same LGR collapse to one node after `uniq()` |
| BLOCKER-4 | BLOCKER | test_SummaryConfigNode.cpp:653 | `lc_multi_record_produces_nodes` asserts `size==2` but dedup reduces to 1; test will fail |
| BLOCKER-5 | BLOCKER | test_SummaryConfigNode.cpp:672 | `lb_multi_record_produces_nodes` asserts `size==2` but dedup reduces to 1; test will fail |
| SUGGESTION-1 | SUGGESTION | SummaryConfig.cpp:891-893 | LGR well-filter logic lacks explanatory comment |
| SUGGESTION-2 | SUGGESTION | SummaryConfig.cpp:885-887, 1466-1468 | `handleMissingWell` fires before LGR filter; diagnostic message could be more targeted |
| SUGGESTION-3 | SUGGESTION | SummaryConfig.hpp:682-685 | `handleProcessingInstruction` doc comment does not mention NOSUMLGR |
| NOTE | NOTE | SummaryConfig.hpp:675 | "Phase 2" is an internal label; prefer external-reviewer-facing wording |
| NOTE | NOTE | SummaryConfig.hpp:165-167 | `{}` zero-init ijk placeholder lacks inline comment explaining deferral |
| NOTE | NOTE | SummaryConfig.cpp:1928-1951 | All LW*/LC*/LB* keywords get `Type::Undefined` from `parseKeywordType`; pre-existing gap, not introduced here |
| NOTE | NOTE | test_SummaryConfigNode.cpp:396-412 | No test exercises NOSUMLGR through `handleProcessingInstruction` path |
| NOTE | NOTE | LGR-WELL-3X3-1LGR.DATA | No deck exercises NOSUMLGR; inline helper is sufficient but deck-level coverage gap exists |

---

## Overall Verdict

**REQUEST CHANGES**

Two categories of blockers must be resolved before merge:

1. **Dispatch correctness (BLOCKER-1):** The `'L'`-prefix dispatch is too coarse and silently misroutes LCOFRU/LCWFRU/LCGFRU and LBPR_X/LBSCN_X variants. Add explicit exclusion sets or tighten the dispatch condition.

2. **IJK deferral creates dedup collapse and failing tests (BLOCKER-3, BLOCKER-4, BLOCKER-5):** Either store I,J,K as `number_` (LGR-local linear index) in `keywordLB` and `keywordLC` so distinct cells are distinguishable after dedup, OR change the test assertions to match the actual one-per-LGR behavior and document the limitation prominently. The `operator==` omission (BLOCKER-2) must also be fixed to match serialization.

The schema files (Section 1), the `is_processing_instruction`/`handleProcessingInstruction` NOSUMLGR integration, the `keywordLW` well-filter logic, the `operator==`/`operator<` LGR extensions in `SummaryConfigNode`, the sorting semantics, and the CMakeLists additions are all correct. The test structure and coverage intent are good once the counting bugs are resolved.
