# Shadow: tests/test_OutputStream.cpp
project: opm-common
indexed: 2026-05-06
updated: 2026-05-06

## Summary
Boost.Test unit tests for all `OutputStream` classes: `Init`, `Restart`,
`RFT`, and `SummarySpecification`. Tests write binary output then read back
with `EclFile` and verify array name, type, element count, and values.

## Test Suites
- `FileName` — `outputFileName()` helper
- `Class_Init` — `Init` write (unformatted, formatted)
- `Class_Restart` — `Restart` write (unified/separate, formatted/unformatted)
- `Class_RFT` — `RFT` write
- `Class_SummarySpecification` — `SummarySpecification::write()` (added 2026-05-06: LGR tests)

## Key Pattern
All tests follow:
1. Construct object → call `write()` → object goes out of scope (file closed)
2. Open the output file with `EclFile` → call `getList()` to verify array sequence
3. Call `loadData()` → `get<T>(name)` to verify values

**CRITICAL — EclFile string readback strips trailing spaces.**
- Blank 8-char entries read back as `""` (not `"        "`)
- `"LGR1    "` reads back as `"LGR1"` (not `"LGR1    "`)
- All `expect` vectors for CHAR arrays must use unpadded strings

## Helper Factories (anonymous namespace at line ~1742)
- `advance()`, `makeUTCTime()` — time construction
- `noWGName()` → `":+:+:+:+"`, `noNum()` → `0`
- `start(year,month,day,hour,min,sec)` → `StartTime`
- `noRestart()` → `RestartSpecification{"", -1}`
- `restartedSimulation()` → 28-char basename, step 123
- `summaryParameters()` → 4 non-LGR vectors (TIME, WBHP, GGOR, BGSAT)
- (no lgrParameters factory added — PR02 tests build params inline)

## PR02 Tests Added (2026-05-06)

### `LGR_Arrays_Written_When_LGR_Vectors_Present`
Setup: TIME + WBHP/PROD01 (non-LGR via nullopt), LWBHP/PROD01 in LGR1 {2,3,4}, LWBHP/PROD02 in LGR1 {5,6,7}
Verifies:
- Array order: `NUMS → LGRS → NUMLX → NUMLY → NUMLZ → UNITS → STARTDAT → LGRNAMES → LGRVEC → LGRTIMES → RUNTIMEI`
- `LGRS` = `["        ", "        ", "LGR1    ", "LGR1    "]`
- `NUMLX` = `[0, 0, 2, 5]`  `NUMLY` = `[0, 0, 3, 6]`  `NUMLZ` = `[0, 0, 4, 7]`
- `LGRNAMES` = `["LGR1    "]` (1 unique name)
- `LGRVEC` = `[4]` (= 2 + 2 appearances of LGR1)
- `LGRTIMES` = `[5]` (currentStep=5)

### `LGR_Arrays_Absent_When_No_LGR_Vectors`
Setup: `summaryParameters()` only (4-arg add, no lgr_info)
Verifies: LGRS, NUMLX, NUMLY, NUMLZ, LGRNAMES, LGRVEC, LGRTIMES all absent from getList()
