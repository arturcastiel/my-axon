# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/io/eclipse/SummaryNode.hpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/io/eclipse/SummaryNode.hpp
shadow-created: 2026-05-05
shadow-updated: 2026-05-07

## Summary
Low-level summary vector identity header. Defines `lgr_info` struct and `SummaryNode` aggregate.
Lives in `opm/io/eclipse/` (EclIO layer — closest to binary SMSPEC/UNSMRY output).

**CANONICAL STATE (revised PR-000, commit 0066645cd): `SummaryNode.hpp` is IDENTICAL TO MASTER.**
No `operator==`, no `serializeOp` added to `lgr_info`. The old PR-000 added these; the revised
PR-000 reverted them. PR-001 (`lgr-summaryconfig`) still carries these additions — must be dropped
in the harmonization rebase.

## Key Structures

### `lgr_info` (struct) — UNCHANGED FROM MASTER
```cpp
struct lgr_info {
    std::string       name {};   // LGR grid name, e.g. "LGR1"
    std::array<int,3> ijk  {};   // 1-based cell coords within the LGR (IO layer only)
};
```
- No `operator==` — config layer uses `lgr_name_` string comparison, not lgr_info comparison
- No `serializeOp` — lgr_info is not serialized in SummaryConfigNode; only `lgr_name_` (string) is
- `ijk` is an IO-layer concern: populated by ESmry.cpp from SMSPEC NUMLX/NUMLY/NUMLZ arrays

### `SummaryNode` (aggregate struct) — UNCHANGED FROM MASTER
Fields: `keyword`, `category` (Category enum), `type` (Type enum), `wgname`, `number`,
        `fip_region` (optional<string>), `lgr` (optional<lgr_info>)

Key static methods (in SummaryNode.cpp):
- `category_from_keyword(kw)` — routes by first char; **PR-000 added `case 'L':`** (in .cpp, not .hpp)
- `normalise_keyword(category, kw)` — strips numeric suffix from well-completion keywords
- `miscellaneous_exception(kw)` — override list for known Misc keywords

### `Category` enum
`Well, Group, Field, Region, Block, Connection, Completion, Segment, Aquifer, Node, Miscellaneous`
- **No `operator<<`** — do NOT use `BOOST_CHECK_EQUAL` for Category comparisons; use `BOOST_CHECK(a == b)`

## Dependencies
- `<array>`, `<functional>`, `<limits>`, `<optional>`, `<string>` — stdlib only
- No opm-common headers included here (base layer)
- Consumed by: `SummaryConfig.hpp`, `SummaryConfig.cpp`, `EclipseIO.cpp`, `SMSPEC` writers, test files

## Architecture Role
Bottom of the LGR summary pipeline. `lgr_info.ijk` is populated by the IO layer (ESmry.cpp reads
NUMLX/NUMLY/NUMLZ from SMSPEC). The config layer (`SummaryConfigNode`) stores only `lgr_name_`
(string) — no ijk, no lgr_info. The conversion operator in `SummaryConfig.hpp` bridges the two:
produces `lgr_info{*lgr_name_, {}}` — ijk placeholder `{}` filled later by PR-002 from SMSPEC arrays.

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR0-old | Old PR-000 added `lgr_info::operator==` (defaulted) and `serializeOp` template |
| 2026-05-07 | PR0-rev | Revised PR-000 (`0066645cd`) reverted both — SummaryNode.hpp now identical to master |
| 2026-05-07 | PR0-rev | Neither addition is needed: config layer compares `lgr_name_` string; lgr_info is not serialized |
| 2026-05-07 | PR1-impact | PR-001 (`lgr-summaryconfig`) still carries these additions — drop in harmonization rebase |
| 2026-05-05 | test | `Category` enum has no `operator<<` — use `BOOST_CHECK(x == Cat::Y)` not `BOOST_CHECK_EQUAL` |
