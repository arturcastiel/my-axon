# PR-001 Review Response Plan — @bska comments

**Branch:** lgr-summaryconfig
**Reviewer:** @bska
**Date:** 2026-05-07
**Status:** PLAN — pending implementation

---

## Reviewer comments mapped to actions

| # | File | Comment | Action |
|---|------|---------|--------|
| R1 | General | Regex too greedy in *_PROBE — vector must not match >1 probe | Tighten regex in LOCAL_WELL_PROBE, LOCAL_CONNECTION_PROBE, LOCAL_BLOCK_PROBE |
| R2 | General | Prefer C++20 constrained range algorithms | Replace `std::find_if/any_of/count_if` → `std::ranges::*` throughout |
| R3 | General | Use raw string literals, not `\n` embedded strings | Replace `parseSummarySection` deck_str with `R"(...)"` |
| R4 | BLOCK_PROBE | LBPR_X/LBSCN_X/LBHD_X/LBHDF_X/LBCTRA_X don't fit the probe format — remove | Remove 5 entries from `BLOCK_PROBE.deck_names` |
| R5 | LOCAL_BLOCK_PROBE | `"LB.+"` matches _X variants — too greedy | Change regex; remove stale comment referencing _X keywords |
| R6 | CONNECTION_PROBE | LCOFRU/LCWFRU/LCGFRU format doesn't match CONNECTION_PROBE — remove | Remove 3 entries from `CONNECTION_PROBE.deck_names` |
| R7 | LOCAL_CONNECTION_PROBE | Comment references LCOFRU etc. which are being removed | Update or remove the comment |
| R8 | SummaryConfig.cpp | `keywordLB` — confirm linearised "number" deferred to follow-up | Add comment; adjust tests to match actual behavior (size==1 per LGR) |
| R9 | SummaryConfig.cpp | `keywordLC` — same question about "number" | Same as R8 |
| R10 | SummaryConfig.cpp | Add braces to single-statement if/else blocks (2 locations) | Add `{}` in Block and Connection dispatch arms |
| R11 | test_SummaryConfigNode.cpp | Use `std::ranges::find_if` | Replace all unconstrained `std::find_if` → `std::ranges::find_if` |
| R12 | test_SummaryConfigNode.cpp | Use raw string literals in `parseSummarySection` | Convert deck_str to `R"(...)"` |
| +1 | SummaryConfig.cpp | `operator==` omits `noSumLgr_` (not from reviewer — correctness fix) | Add `noSumLgr_` to `SummaryConfig::operator==` |

---

## Step-by-step implementation plan

---

### Step 1 — Remove LB*_X from BLOCK_PROBE
**File:** `opm/input/eclipse/share/keywords/000_Eclipse100/B/BLOCK_PROBE`

Remove from `deck_names`:
```
"LBCTRA_X"
"LBHDF_X"
"LBHD_X"
"LBPR_X"
"LBSCN_X"
```
These require a different item set than BLOCK_PROBE's `(I, J, K)`.
They are unsupported for now and should not appear in any probe.

**Verification:** `grep -c "LBPR_X\|LBSCN_X\|LBHD_X\|LBHDF_X\|LBCTRA_X" BLOCK_PROBE` = 0

---

### Step 2 — Remove LCOFRU/LCWFRU/LCGFRU from CONNECTION_PROBE
**File:** `opm/input/eclipse/share/keywords/000_Eclipse100/C/CONNECTION_PROBE`

Remove from `deck_names`:
```
"LCOFRU"
"LCWFRU"
"LCGFRU"
```
Their format does not match CONNECTION_PROBE's `(WELL, I, J, K)` item set.

Note: `COFRU`, `CWFRU`, `CGFRU` (without L prefix) stay — they are global connection vectors.

**Verification:** `grep -c "LCOFRU\|LCWFRU\|LCGFRU" CONNECTION_PROBE` = 0

---

### Step 3 — Tighten regex in LOCAL_WELL_PROBE
**File:** `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_WELL_PROBE`

Change:
```json
"deck_name_regex": "LW.+"
```
To:
```json
"deck_name_regex": "LW[A-Z]+"
```
`[A-Z]+` restricts to uppercase letters only — excludes `_X` and numeric suffix patterns.

---

### Step 4 — Tighten regex + update comment in LOCAL_CONNECTION_PROBE
**File:** `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_CONNECTION_PROBE`

Change regex:
```json
"deck_name_regex": "LC.+"
```
To:
```json
"deck_name_regex": "LC[A-Z]+"
```

Update comment — remove reference to LCOFRU/LCWFRU/LCGFRU (now removed from CONNECTION_PROBE and not yet placed anywhere):
```json
"comment": "LGR connection-level summary vectors (LC*)."
```

---

### Step 5 — Tighten regex + update comment in LOCAL_BLOCK_PROBE
**File:** `opm/input/eclipse/share/keywords/000_Eclipse100/L/LOCAL_BLOCK_PROBE`

Change regex:
```json
"deck_name_regex": "LB.+"
```
To:
```json
"deck_name_regex": "LB[A-Z]+"
```

Update comment — remove reference to the _X keywords (now removed from BLOCK_PROBE):
```json
"comment": "LGR block-level summary vectors (LB*)."
```

---

### Step 6 — Add braces to dispatch if/else blocks
**File:** `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp`

Location ~1789 (Block dispatch):
```cpp
// Before:
if (keyword.name()[0] == 'L')
    keywordLB(list, keyword);
else
    keywordB(list, keyword, dims);

// After:
if (keyword.name()[0] == 'L') {
    keywordLB(list, keyword);
}
else {
    keywordB(list, keyword, dims);
}
```

Location ~1800 (Connection dispatch) — same pattern for `keywordLC` / `connectionKeyword`.

---

### Step 7 — Document deferred number in keywordLB and keywordLC
**File:** `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp`

In `keywordLB`, before `list.push_back`:
```cpp
// number_ (linearised cell index within the LGR) is deferred to follow-up work.
// All LB* records for the same LGR currently share number_=INT_MIN.
list.push_back(param.lgr_name(lgr_name));
```

Same comment in `keywordLC` before its `list.push_back`.

---

### Step 8 — Fix `SummaryConfig::operator==` to include `noSumLgr_`
**File:** `opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.cpp`

Location ~2503:
```cpp
// Before:
return (this->m_keywords     == data.m_keywords)
    && (this->short_keywords  == data.short_keywords)
    && (this->summary_keywords == data.summary_keywords)
    ;

// After:
return (this->m_keywords      == data.m_keywords)
    && (this->short_keywords   == data.short_keywords)
    && (this->summary_keywords == data.summary_keywords)
    && (this->noSumLgr_        == data.noSumLgr_)
    ;
```

---

### Step 9 — Replace unconstrained algorithms with std::ranges in test file
**File:** `tests/test_SummaryConfigNode.cpp`

Replace all occurrences:
```cpp
// Before:
std::find_if(hits.begin(), hits.end(), pred)
std::any_of(hits.begin(), hits.end(), pred)
std::count_if(hits.begin(), hits.end(), pred)

// After:
std::ranges::find_if(hits, pred)
std::ranges::any_of(hits, pred)
std::ranges::count_if(hits, pred)
```

Locations: `lgr_1lgr_global_well_has_no_lgr`, `lgr_1lgr_lb_node_populated`,
`lgr_2lgr_lw_two_nodes_not_deduped`, `lgr_2lgr_lb_dedup_by_lgr_name`.

---

### Step 10 — Replace escaped string literals with raw string literals
**File:** `tests/test_SummaryConfigNode.cpp`

The `parseSummarySection` helper (~line 559):

```cpp
// Before:
const std::string deck_str =
    "RUNSPEC\n"
    "TITLE\n"
    " LGR_SCHEMA_TEST /\n"
    "DIMENS\n"
    " 3 3 1 /\n"
    ...

// After:
const std::string deck_str = R"(
RUNSPEC
TITLE
 LGR_SCHEMA_TEST /
DIMENS
 3 3 1 /
WELLDIMS
 4 1 1 4 /
START
 1 JAN 2020 /
GRID
DXV
 3*100.0 /
DYV
 3*100.0 /
DZV
 1*10.0 /
DEPTHZ
 16*2000.0 /
PORO
 9*0.3 /
PROPS
SOLUTION
SUMMARY
)" + summary_body + R"(
SCHEDULE
WELSPECL
 'WELL1'  'G'  'LGR1'  2 2  2000.0  'OIL' /
 'WELL2'  'G'  'LGR2'  3 3  2000.0  'OIL' /
 'INJ'    'G'  'LGR1'  1 1  2000.0  'WAT' /
 'PROD'   'G'  'LGR2'  3 3  2000.0  'OIL' /
/
DATES
 1 FEB 2020 /
/
END
)";
```

Also check test deck inline strings in `LGR_SummaryConfig_Deck` cases for any `\n` escapes.

---

### Step 11 — Fix test count assertions for same-LGR multi-record cases
**File:** `tests/test_SummaryConfigNode.cpp`

Because `keywordLB` and `keywordLC` defer `number_` (Step 7), two records for
the same LGR produce equal nodes and deduplicate to 1 after `uniq()`.

`lc_multi_record_produces_nodes` (~line 653):
```cpp
// Before:
BOOST_CHECK_EQUAL(nodes.size(), 2u);

// After: same-well + same-LGR LC records collapse to 1 (number_ deferred)
BOOST_CHECK_EQUAL(nodes.size(), 1u);
```

`lb_multi_record_produces_nodes` (~line 672):
```cpp
// Before:
BOOST_REQUIRE_EQUAL(nodes.size(), 2u);

// After: same-LGR LB records collapse to 1 (number_ deferred)
BOOST_REQUIRE_EQUAL(nodes.size(), 1u);
```

Add brief comments documenting why (deferred number_ → dedup).

---

### Step 12 — Add NOSUMLGR deck-parsing test
**File:** `tests/test_SummaryConfigNode.cpp`

Add to `NoSumLgr` suite, using `parseSummarySection` (already defined in `LGR_Schema_Inline`):
```cpp
BOOST_AUTO_TEST_CASE(deck_sets_flag)
{
    const auto cfg = parseSummarySection("NOSUMLGR\n");
    BOOST_CHECK(cfg.noSumLgr());
}
```

Note: `parseSummarySection` is in an anonymous namespace inside `LGR_Schema_Inline` — the `NoSumLgr` suite is declared before it. Either move `parseSummarySection` earlier, or add a simpler inline deck inside `NoSumLgr`.

---

### Step 13 — Build and test
```bash
cmake --build /home/arturcastiel/opm-hd/projects/opm-development/build/opm-common \
    --target test_SummaryConfigNode opmcommon -- -j$(nproc)

ctest --test-dir /home/arturcastiel/opm-hd/projects/opm-development/build/opm-common \
    -R "^(SummaryNode|SummaryConfigNode|SummaryConfigTests|Serialization)$" \
    --output-on-failure
```

Note: `SummaryConfigNode` (#77) is the target that runs `test_SummaryConfigNode.cpp`.
It was missing from previous ctest runs — must be included.

---

### Step 14 — Commit
All changes are fixups to existing commits. Add as a single review-response commit:
```
SummaryConfig: address @bska review comments

- Tighten deck_name_regex in LOCAL_*_PROBE (LW/LC/LB[A-Z]+)
- Remove LB*_X from BLOCK_PROBE and LCOFRU/LCWFRU/LCGFRU from
  CONNECTION_PROBE (unsupported — different item format)
- Add braces around single-statement if/else in dispatch
- Document deferred number_ in keywordLB and keywordLC; adjust
  test counts to match actual dedup behaviour
- Add noSumLgr_ to SummaryConfig::operator==
- Use std::ranges algorithms throughout test file
- Convert escaped string literal to R"(...)" in parseSummarySection
- Add NOSUMLGR deck-parsing test case
```

---

## Risk assessment

| Step | Risk | Note |
|---|---|---|
| 1-2 | Low | Removing entries from existing probe files — no logic change |
| 3-5 | Low | Regex tightening — `[A-Z]+` strictly excludes `_X` and numeric suffixes |
| 6 | Trivial | Style only |
| 7 | Low | Comment addition only |
| 8 | Medium | `operator==` fix — regression possible if equality semantics differ |
| 9 | Low | Direct API substitute — same semantics |
| 10 | Low-Medium | Raw string concat with `+ summary_body +` must preserve section ordering |
| 11 | Medium | Changing assertions from 2→1 — confirms dedup rather than fixing it |
| 12 | Low | New test case |
| Step 12 note | Medium | `parseSummarySection` scope — may need forward-declare or restructure |
