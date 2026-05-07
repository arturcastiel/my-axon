# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.hpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/input/eclipse/EclipseState/SummaryConfig/SummaryConfig.hpp
shadow-created: 2026-05-05
shadow-updated: 2026-05-06
git-hash: lgr-summary branch (PR-000 revised)

## Summary
Mid-layer header: defines `SummaryConfigNode` (the high-level summary vector definition used by the input parser) and the free functions `parseKeywordCategory`, `parseKeywordType`, `operator==`, `operator<`.

**PR-000 revised design** (reviewer @bska feedback): replaced `optional<EclIO::lgr_info> lgr_` with two cleaner fields:
- `optional<string> lgr_name_` — LGR grid name (always set for LGR nodes)
- `lgr_index_` dropped entirely — `number_` already carries numeric identity; LGR ordering uses string compare on `lgr_name_`

## Key Structures

### `SummaryConfigNode` class
Builder pattern with fluent setters (return `*this`):
- `parameterType(Type)` → sets `type_`
- `namedEntity(string)` → sets `name_`
- `number(int)` → sets `number_`
- `isUserDefined(bool)` → sets `userDefined_`
- `fip_region(string)` → sets `fip_region_`
- `lgr_name(string)` → sets `lgr_name_`  **PR0 ADDITION (revised)**

Getters: `keyword()`, `category()`, `type()`, `namedEntity()`, `number()`, `isUserDefined()`, `fip_region()`, `lgr_name()` **PR0 ADDITION (revised)**, `uniqueNodeKey()`, `location()`

### `operator EclIO::SummaryNode()` — conversion operator (PR0 MODIFIED)
```cpp
operator EclIO::SummaryNode() const {
    auto lgr = this->lgr_name_.has_value()
        ? std::optional<EclIO::lgr_info>{ EclIO::lgr_info{ *this->lgr_name_, {} } }
        : std::optional<EclIO::lgr_info>{};
    return {
        this->keyword_, this->category_, this->type_,
        this->name_,    this->number_,   this->fip_region_,
        std::move(lgr)  // PR0: named local avoids aggregate-init ternary ambiguity
    };
}
```
Critical: ternary expression inline in brace-init list causes compiler error (aggregate deduction failure). Must use named local variable.

### `serializeOp()` (PR0 MODIFIED)
```cpp
template<class Serializer>
void serializeOp(Serializer& serializer) {
    serializer(keyword_);
    serializer(category_);
    serializer(loc);
    serializer(type_);
    serializer(name_);
    serializer(number_);
    serializer(fip_region_);
    serializer(userDefined_);
    serializer(lgr_name_);   // PR0 ADDITION (revised) — MUST be last (binary layout)
}
```
**CRITICAL**: `serializer(lgr_name_)` must remain LAST. Preserves binary compat with existing restart files.

### Private fields
```
keyword_     string
category_    Category
type_        Type (default Undefined)
name_        string
number_      int (default numeric_limits<int>::min())
fip_region_  optional<string>
userDefined_ bool (default false)
loc          KeywordLocation
lgr_name_    optional<string>   // PR0 ADDITION (revised) — LGR grid name
```

### Free functions (declared here, defined in SummaryConfig.cpp)
- `parseKeywordCategory(kw)` → Category  — SEPARATE from `SummaryNode::category_from_keyword`
- `parseKeywordType(kw)` → Type
- `operator==(lhs, rhs)` — compares `lgr_name()` for all categories (PR0 revised)
- `operator<(lhs, rhs)` — tiebreaker: `lgr_name().has_value()` check then `*lgr_name()` string compare (PR0 revised; no lgr_index)

## Design Notes — PR-000 revision
- `lgr_info` struct (with ijk array) lives only in `SummaryNode.hpp` / ESmry.cpp (IO layer). Config layer never sees ijk.
- `lgr_index_` was considered (LGR declaration order from `EclipseGrid::get_lgr_cell_index`), but dropped: `number_` already carries numeric identity per node; string compare on name is sufficient for `operator<` and avoids the ijk ordering bug @bska flagged.
- PR-001 will call `.lgr_name(lgr_name)` in `keywordLW`, `keywordLC`, `keywordLB`.

## Dependencies
- `<opm/io/eclipse/SummaryNode.hpp>` — for `EclIO::SummaryNode` Category/Type enums (no `lgr_info` usage)
- `<opm/common/OpmLog/KeywordLocation.hpp>`
- `<array>`, `<cstddef>`, `<limits>`, `<optional>`, `<set>`, `<string>`, `<unordered_set>`, `<vector>`
- Forward declares: `AquiferConfig`, `Deck`, `EclipseState`, `ErrorGuard`, `FieldPropsManager`, `GridDims`, `ParseContext`, `Schedule`

## Architecture Role
Middle of the three-layer LGR pipeline. `SummaryConfigNode` is the object created by the input parser (SummaryConfig.cpp) when it processes SUMMARY section keywords. It is later converted to `EclIO::SummaryNode` for the binary output layer. `lgr_name_` flows from the parser (PR-001) through this class into the SMSPEC/UNSMRY writers (PR-002+).

## Findings Log
| date | context | finding |
|------|---------|----------|
| 2026-05-05 | PR0 | Added `optional<EclIO::lgr_info> lgr_{}` private field |
| 2026-05-05 | PR0 | Added `lgr()` const getter returning `const optional<lgr_info>&` |
| 2026-05-05 | PR0 | Added `lgr(const lgr_info&)` fluent setter returning `*this` |
| 2026-05-05 | PR0 | `operator EclIO::SummaryNode()` updated to pass `this->lgr_` as last positional arg |
| 2026-05-05 | PR0 | `serializer(lgr_)` appended LAST in `serializeOp()` — binary layout constraint |
| 2026-05-06 | PR0-rev | Replaced `optional<lgr_info> lgr_` with `optional<string> lgr_name_` (reviewer @bska: ijk in config layer is wrong) |
| 2026-05-06 | PR0-rev | Dropped `lgr_index_` (was `optional<size_t>` for declaration-order sort) — redundant: `number_` carries numeric identity; string compare on `lgr_name_` sufficient for `operator<` |
| 2026-05-06 | PR0-rev | `operator<` now ends with `return *lhs.lgr_name() < *rhs.lgr_name()` — no ijk, no declaration-index |
| 2026-05-06 | PR0-rev | `SummaryNode::lgr` field renamed to `lgr_grid` (optional<string>) to match this layer |

## Summary
Mid-layer header: defines `SummaryConfigNode` (the high-level summary vector definition used by the input parser) and the free functions `parseKeywordCategory`, `parseKeywordType`, `operator==`, `operator<`. PR0 added: `lgr_` private field, `lgr()` getter, `lgr(info)` setter (fluent), `lgr_` in `operator EclIO::SummaryNode()`, and `serializer(lgr_)` at end of `serializeOp()`.

## Key Structures

### `SummaryConfigNode` class
Builder pattern with fluent setters (return `*this`):
- `parameterType(Type)` → sets `type_`
- `namedEntity(string)` → sets `name_`
- `number(int)` → sets `number_`
- `isUserDefined(bool)` → sets `userDefined_`
- `fip_region(string)` → sets `fip_region_`
- `lgr(lgr_info)` → sets `lgr_`  **PR0 ADDITION**

Getters: `keyword()`, `category()`, `type()`, `namedEntity()`, `number()`, `isUserDefined()`, `fip_region()`, `lgr()` **PR0 ADDITION**, `uniqueNodeKey()`, `location()`

### `operator EclIO::SummaryNode()` — conversion operator (PR0 MODIFIED)
```cpp
operator EclIO::SummaryNode() const {
    return {
        this->keyword_,
        this->category_,
        this->type_,
        this->name_,
        this->number_,
        this->fip_region_,
        this->lgr_          // PR0 ADDITION: passes lgr_ into low-level node
    };
}
```
Critical: `SummaryNode` aggregate initializer order must match field declaration order in `SummaryNode.hpp`.

### `serializeOp()` (PR0 MODIFIED)
```cpp
template<class Serializer>
void serializeOp(Serializer& serializer) {
    serializer(keyword_);
    serializer(category_);
    serializer(loc);
    serializer(type_);
    serializer(name_);
    serializer(number_);
    serializer(fip_region_);
    serializer(userDefined_);
    serializer(lgr_);        // PR0 ADDITION — MUST be last (binary layout)
}
```
**CRITICAL**: `serializer(lgr_)` must remain LAST. Adding it last preserves binary compatibility with existing restart files that don't contain LGR data.

### Private fields
```
keyword_     string
category_    Category
type_        Type (default Undefined)
name_        string
number_      int (default numeric_limits<int>::min())
fip_region_  optional<string>
userDefined_ bool (default false)
loc          KeywordLocation
lgr_         optional<EclIO::lgr_info>   // PR0 ADDITION
```

### Free functions (declared here, defined in SummaryConfig.cpp)
- `parseKeywordCategory(kw)` → Category  — SEPARATE from `SummaryNode::category_from_keyword`
- `parseKeywordType(kw)` → Type
- `operator==(lhs, rhs)` — must include lgr comparison for Well/Block/Connection categories (PR0)
- `operator<(lhs, rhs)` — tiebreaker chain ending in lgr comparison (PR0)

## Dependencies
- `<opm/io/eclipse/SummaryNode.hpp>` — for `EclIO::SummaryNode`, `EclIO::lgr_info`
- `<opm/common/OpmLog/KeywordLocation.hpp>`
- `<array>`, `<cstddef>`, `<limits>`, `<optional>`, `<set>`, `<string>`, `<unordered_set>`, `<vector>`
- Forward declares: `AquiferConfig`, `Deck`, `EclipseState`, `ErrorGuard`, `FieldPropsManager`, `GridDims`, `ParseContext`, `Schedule`

## Architecture Role
Middle of the three-layer LGR pipeline. `SummaryConfigNode` is the object created by the input parser (SummaryConfig.cpp) when it processes SUMMARY section keywords. It is later converted to `EclIO::SummaryNode` for the binary output layer. The `lgr_` field added in PR0 is the identity carrier that flows from the parser (PR01) through this class into the SMSPEC/UNSMRY writers (PR02+).

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR0 | Added `optional<EclIO::lgr_info> lgr_{}` private field |
| 2026-05-05 | PR0 | Added `lgr()` const getter returning `const optional<lgr_info>&` |
| 2026-05-05 | PR0 | Added `lgr(const lgr_info&)` fluent setter returning `*this` |
| 2026-05-05 | PR0 | `operator EclIO::SummaryNode()` updated to pass `this->lgr_` as last positional arg |
| 2026-05-05 | PR0 | `serializer(lgr_)` appended LAST in `serializeOp()` — binary layout constraint |
| 2026-05-05 | design | `parseKeywordCategory()` in SummaryConfig.cpp is SEPARATE from `category_from_keyword()` in SummaryNode.cpp — both need `case 'L':` but PR01 touches parseKeywordCategory |
