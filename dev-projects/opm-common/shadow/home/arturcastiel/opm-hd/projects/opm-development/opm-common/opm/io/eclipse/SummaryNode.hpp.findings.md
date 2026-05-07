# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/io/eclipse/SummaryNode.hpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/io/eclipse/SummaryNode.hpp
shadow-created: 2026-05-05
shadow-updated: 2026-05-05
git-hash: a200fcf850333daf5e2adc63829fdf46c3c654f4b203935aae6fc230ed0b5e3c

## Summary
Low-level summary vector identity header. Defines `lgr_info` struct and `SummaryNode` aggregate. Lives in `opm/io/eclipse/` (EclIO layer — closest to binary SMSPEC/UNSMRY output). PR-000 added `operator==` and `serializeOp` to the pre-existing `lgr_info` struct. **`SummaryNode` itself is unchanged from master** — it still has `optional<lgr_info> lgr {}`.

NOTE: At one point a stale commit attempted to rename `lgr` → `lgr_grid` (optional<string>); this was an error and was corrected. The canonical state is `optional<lgr_info> lgr{}` with lgr_info having operator== and serializeOp.

## Key Structures

### `lgr_info` (struct)
```cpp
struct lgr_info {
    std::string    name {};        // LGR grid name, e.g. "LGR1"
    std::array<int,3> ijk {};      // 1-based cell coords within the LGR

    bool operator==(const lgr_info&) const = default;  // PR0: C++20 defaulted

    template<class Serializer>
    void serializeOp(Serializer& s) { s(name); s(ijk); }  // PR0: serialization
};
```
- `operator==` defaulted — memberwise equality, used by ESmry.cpp comparison
- `serializeOp` template — defensive; `EclIO::SummaryNode` is not directly serialized in any test but keeps the type serialization-capable

### `SummaryNode` (aggregate struct) — **UNCHANGED FROM MASTER**
Fields: `keyword`, `category` (Category enum), `type` (Type enum), `wgname`, `number`, `fip_region` (optional<string>), `lgr` (optional<lgr_info>)

Key static methods:
- `category_from_keyword(kw)` — routes by first char; **PR0 added `case 'L':`**
- `normalise_keyword(category, kw)` — strips numeric suffix from well-completion keywords
- `miscellaneous_exception(kw)` — override list for known Misc keywords

### `Category` enum
`Well, Group, Field, Region, Block, Connection, Completion, Segment, Aquifer, Node, Miscellaneous`
- **No `operator<<`** — do NOT use `BOOST_CHECK_EQUAL` for Category comparisons in tests; use `BOOST_CHECK(a == b)`

## Dependencies
- `<array>`, `<functional>`, `<limits>`, `<optional>`, `<string>` — stdlib only
- No opm-common headers included here (base layer)
- Consumed by: `SummaryConfig.hpp`, `SummaryConfig.cpp`, `EclipseIO.cpp`, `SMSPEC` writers, test files

## Architecture Role
Bottom of the LGR summary pipeline. `lgr_info` defined here propagates upward through `SummaryConfigNode` (mid layer) to SMSPEC/UNSMRY writers (top layer). Changes here affect everything above.

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR0 | Added `lgr_info` struct with `operator==` (defaulted) and `serializeOp` template |
| 2026-05-05 | PR0 | Added `std::optional<lgr_info> lgr {}` field to `SummaryNode` aggregate |
| 2026-05-05 | PR0 | `case 'L':` routing logic lives in `SummaryNode.cpp`, not this header |
| 2026-05-05 | test | `Category` enum has no `operator<<` — Boost.Test `BOOST_CHECK_EQUAL` fails at compile time; use `BOOST_CHECK(x == Cat::Y)` |
