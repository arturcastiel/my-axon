# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/CMakeLists_files.cmake
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/CMakeLists_files.cmake
shadow-created: 2026-05-05
shadow-updated: 2026-05-05
git-hash: (not hashed — large generated file, findings only)

## Summary
Central list file for opm-common build system. Declares all source files, test files, and data files consumed by the cmake build. Three main list variables used by the opm cmake macros.

## Key Structures

### `OPMCOMMON_SOURCE_FILES`
All `.cpp` files compiled into the `opmcommon` library. Not relevant for test data.

### `OPMCOMMON_TEST_SUITE_FILES`
All test `.cpp` files. Each entry becomes a separate CTest test executable via `opm_add_test`.
Relevant entries:
- `tests/test_SummaryConfigNode.cpp` (line 560) — PR01 tests go here
- `tests/test_Summary.cpp` (line 558)
- `tests/test_SummaryNode.cpp` (line 559)

### `OPMCOMMON_DATA_FILES`
All data files (`.DATA`, `.inc`, `.py`, `.json`, etc.) that must be available in
`${PROJECT_BINARY_DIR}/tests/` at test run time. CMake symlinks (or copies) them from
the source tree to the build tree during configure.

**The mechanism:** cmake iterates this list and calls `configure_file` or
`file(CREATE_LINK ...)` for each entry, placing the file at
`${PROJECT_BINARY_DIR}/{relative-path}`.

Relevant entries (lines 834-835):
```
tests/summary_deck.DATA
tests/summary_deck_non_constant_porosity.DATA
```

**To add a new test .DATA file:** append to `OPMCOMMON_DATA_FILES` in this file.
The path must be relative to the opm-common source root (e.g. `tests/LGR-WELL-3X3-1LGR.DATA`).

## Dependencies
Consumed by `CMakeLists.txt` via `include(CMakeLists_files.cmake)`.

## Architecture Role
Single source of truth for which files the build system knows about. Any test `.DATA` file not listed here will NOT be available in the build-tree test directory — the test will fail with "file not found" at runtime even if the file is in the source tree.

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR01 study | test_SummaryConfigNode.cpp is at line 560 of OPMCOMMON_TEST_SUITE_FILES. No special WORKING_DIRECTORY override — uses default ${PROJECT_BINARY_DIR}/tests. |
| 2026-05-05 | PR01 study | test_Summary.cpp uses `Parser{}.parseFile("summary_deck.DATA")` — works because summary_deck.DATA is in OPMCOMMON_DATA_FILES and gets symlinked to ${PROJECT_BINARY_DIR}/tests/. |
| 2026-05-05 | PR01 ✓ | Added `tests/LGR-WELL-3X3-1LGR.DATA` and `tests/LGR-WELL-3X3-2LGR.DATA` to OPMCOMMON_DATA_FILES (lines 724-725, before `tests/LGR_3WELLS.DATA` — hyphen < underscore in ASCII) |
| 2026-05-06 | keyword_list.cmake | Separate file: `opm/input/eclipse/share/keywords/keyword_list.cmake`. Added 4 entries: `000_Eclipse100/L/LOCAL_BLOCK_PROBE` (after LKRWR), `LOCAL_CONNECTION_PROBE`, `LOCAL_WELL_PROBE`, `000_Eclipse100/N/NOSUMLGR` (after NOSIM). Missing entries → "Unknown keyword" errors at parse time. |
| 2026-05-05 | PR01 test-plan | Must add `tests/LGR-WELL-3X3-1LGR.DATA` and `tests/LGR-WELL-3X3-2LGR.DATA` to OPMCOMMON_DATA_FILES. BUT these files live in opm-tests/summary/, not in opm-common/tests/. Two options: (A) copy/symlink them into opm-common/tests/ and add to OPMCOMMON_DATA_FILES, or (B) use an absolute path via the opm-tests cmake integration (Findopm-tests.cmake at CMakeLists.txt:481). Option A is simpler and consistent with existing pattern. |
| 2026-05-05 | PR01 test-plan | HUMAN INPUT NEEDED: confirm approach — copy .DATA files into opm-common/tests/ and add to CMakeLists_files.cmake, OR reference via opm-tests path. |
