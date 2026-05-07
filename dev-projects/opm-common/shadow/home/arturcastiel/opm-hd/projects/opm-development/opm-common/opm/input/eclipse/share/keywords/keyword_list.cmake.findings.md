# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/keyword_list.cmake
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/share/keywords/keyword_list.cmake
shadow-created: 2026-05-06
shadow-updated: 2026-05-06
pr: PR01

## Summary
Master list of all keyword JSON description files loaded by the OPM parser. One entry per keyword JSON file, using paths relative to `opm/input/eclipse/share/keywords/`. Must be kept in strict alphabetical order. Entries missing from this list cause "Unknown keyword: XXXX" errors at parse time — the parser never reads the corresponding JSON file.

## Key Facts
- Adding a new keyword JSON file is not enough — it must also be listed here
- Order: strictly alphabetical, A-Z, number/hyphen < letter in ASCII (so `LGR-X` before `LGR_Y`)
- File lives at: `opm/input/eclipse/share/keywords/keyword_list.cmake`
- Not to be confused with `CMakeLists_files.cmake` (which lists C++ source/test/data files)
- Missing entries → "Unknown keyword" errors that only appear when the parser encounters the keyword at runtime (e.g. in a .DATA file or inline deck test)

## PR01 Additions (4 entries added)

```cmake
# After LKRWR, before LOAD:
000_Eclipse100/L/LOCAL_BLOCK_PROBE
000_Eclipse100/L/LOCAL_CONNECTION_PROBE
000_Eclipse100/L/LOCAL_WELL_PROBE

# After NOSIM, before NOWARN:
000_Eclipse100/N/NOSUMLGR
```

## Jenkins Failure Caused by Missing Entries
Jenkins failure "Unknown keyword: LWBHP" traced to `LOCAL_WELL_PROBE` (and siblings) not being in this file. The keyword JSON files existed on disk but the parser had no knowledge of them. Adding all 4 entries to the list fixed the "Unknown keyword" error.

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-06 | Jenkins failure #1 | "Unknown keyword: LWBHP" — LOCAL_WELL_PROBE/CONNECTION_PROBE/BLOCK_PROBE/NOSUMLGR all absent from keyword_list.cmake |
| 2026-05-06 | PR01 ✓ | Added 4 entries at correct alphabetical positions; squashed into commit 1 (62085e8d8) |
| 2026-05-06 | lesson | Always add keyword_list.cmake entry alongside the JSON file — adding only the JSON is silent at cmake time, fails at parse time |
