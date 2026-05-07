# Shadow: opm/io/eclipse/OutputStream.cpp
project: opm-common
indexed: 2026-05-05
updated: 2026-05-06

## Summary
Implementation of the Eclipse binary output stream layer. Contains all
SummarySpecification method bodies, helper functions for SMSPEC array
construction (makeDimens, makeStartDate, makeRuntimei, restartRoot, etc.),
and factory functions for creating output files.

## Key Functions

### `Parameters::add()` (line ~744)
```cpp
void Parameters::add(const std::string& keyword, const std::string& wgname,
                     const int num, const std::string& unit)
{
    this->keywords.emplace_back(keyword);
    this->wgnames .emplace_back(wgname);
    this->nums    .push_back(num);
    this->units   .emplace_back(unit);
}
```
4-arg only. All 4 vectors are appended atomically.

### `SummarySpecification::write()` (line ~806)
Current array write order (post PR02 fixes, final clean state 2026-05-06):
1. `INTEHEAD` — `{unit_, 100}`
2. `RESTART`  — restart root (always written; even if empty)
3. `DIMENS`   — `{nparam, nx, ny, nz, 0, istart}`
4. `KEYWORDS` — `params.keywords`
5. `WGNAMES`  — `params.wgnames`
6. `NUMS`     — `params.nums`
7. **[conditional LGR block]** `LGRS` → `NUMLX` → `NUMLY` → `NUMLZ`
8. `UNITS`    — `params.units`
9. `STARTDAT` — 6-element int vector `{day, month, year, hour, min, microsec}`
10. **[conditional LGR block]** `LGRNAMES` → `LGRVEC` → `LGRTIMES`
11. `RUNTIMEI` — 50-element int vector (simulation status, current step, timestamps)

**PR02 fixes applied:**
- Fix 1: LGRS/NUMLX/NUMLY/NUMLZ moved from after-STARTDAT to after-NUMS (before UNITS) ✓
- Fix 2: LGRVEC added after LGRNAMES: `2 + count_if(params.lgrs, matches name)` ✓
- Fix 3: LGRTIMES added after LGRVEC: `currentStep` per LGR (approximation) ✓

**Implementation notes:**
- Blank check: `std::string(s.c_str()) != "        "` — used for `has_lgr` guard and LGRNAMES filter
- `PaddedOutputString<8>` has no `operator==` or `operator<` — all comparisons via `std::string(s.c_str())`
- LGRNAMES: collected into `lgrname_strs` (`std::vector<std::string>`), sorted+deduped as strings, then converted back to `PaddedOutputString<8>` for write via `lgrnames.emplace_back(name)`
- LGRVEC inner comparison: `std::string(s.c_str()) == lgr_name` (lgr_name is already a `std::string`)
- `lgrname_strs` and `lgrnames` both scoped inside `if (has_lgr)` block; `lgrnames` declared before block, populated inside

**Tests added (2026-05-06):** `test_OutputStream.cpp` — `LGR_Arrays_Written_When_LGR_Vectors_Present`, `LGR_Arrays_Absent_When_No_LGR_Vectors`
- `LGR_Arrays_Written_When_LGR_Vectors_Present`: TIME+WBHP (non-LGR) + LWBHP×2 in LGR1; verifies array order, LGRS blanks, NUMLX/Y/Z values, LGRNAMES=["LGR1"], LGRVEC=[4] (2+2), LGRTIMES=[5]
- `LGR_Arrays_Absent_When_No_LGR_Vectors`: uses `summaryParameters()` only; asserts all 7 LGR arrays absent

### `<algorithm>` already included (line 28)
`std::sort`, `std::unique`, `std::any_of` are available without new includes.

## Key Facts for PR02
| Fact | Detail |
|---|---|
| `<algorithm>` included | Line 28 — `std::sort`, `std::unique`, `std::any_of` available |
| Correct position for LGRS/NUMLX/NUMLY/NUMLZ | After NUMS, **before UNITS** (reference-confirmed) |
| Correct position for LGRNAMES/LGRVEC/LGRTIMES | After STARTDAT, before RUNTIMEI (reference-confirmed) |
| Conditional guard | Write LGR per-vector arrays only when `!lgrs.empty() && any_of(not-blank)` |
| LGRNAMES derivation | Sorted unique non-blank entries from `params.lgrs` — correct as-is |
| LGRVEC semantics | `{count}` per LGR = 2 + number of params.lgrs entries matching that LGR name; the +2 is for TIME+YEARS always present in ROOT.LGR file (FileFormatsReferenceManual p.11) |
| LGRTIMES semantics | `{currentStep}` per LGR = total local ministeps written so far; use `currentStep` as approximation |
| `PaddedOutputString<8>` blank | Default-constructed = 8 spaces; compare as `std::string` == `"        "` |
| `rewindStream()` + `flushStream()` | Called at start/end of `write()` — LGR block goes in between |

## Correct Target write() Array Order (post-fix)
```
INTEHEAD → RESTART → DIMENS → KEYWORDS → WGNAMES → NUMS
→ [if LGR: LGRS NUMLX NUMLY NUMLZ]
→ UNITS → STARTDAT
→ [if LGR: LGRNAMES LGRVEC LGRTIMES]
→ RUNTIMEI
```

## PR02 Status: COMPLETE (c87544211)
All 3 fixes applied. Tests added. Ready to push.

## Deferred (separate PR, non-LGR-blocking)
- MEASRMNT: per-vector 32-char description (4×N chars) — pre-existing OPM gap
- RUNTIMED: 5 doubles at end — pre-existing OPM gap
