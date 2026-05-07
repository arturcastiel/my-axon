# Shadow: opm/input/eclipse/EclipseState/Grid/EclipseGrid.hpp
project: opm-common
indexed: 2026-05-06

## Summary
`EclipseGrid` — the grid data structure. Stores cell geometry, active cell index,
LGR sub-grid structure, and LGR label management.


EclipseGrid header — main grid class for global and LGR grids. Extends GridDims. Key LGR identity pattern: get_lgr_cell_index(lgr_tag) returns 0-based index of an LGR by name within all_lgr_labels (offset by 1 because index 0 = GLOBAL). get_lgr_labels_by_number(num) is the inverse. all_lgr_labels[0] = 'GLOBAL'; LGR names start at index 1. lgr_info in SummaryNode uses name+ijk; EclipseGrid resolves name→index via get_lgr_cell_index. The reviewer bska wants SummaryConfigNode to use optional<string> lgr_name + optional<size_t> lgr_number (index), dropping the ijk array from SummaryConfigNode entirely — ijk to number_ conversion happens via getActiveIndexLGR at evaluator time.
## LGR-relevant API

### `get_all_lgr_labels()` (line 136)
```cpp
std::vector<std::string> get_all_lgr_labels() const
```
Returns LGR names in the grid. **Index 0 is the global grid — skip it.** Real LGR
names start at index 1.
```cpp
// Correct usage:
const auto labels = grid.get_all_lgr_labels(); // skip index 0
```
The internal `all_lgr_labels` stores global grid at [0]; the public method strips it.

### `getActiveIndexLGR()` (line 103)
```cpp
size_t getActiveIndexLGR(const std::string& label, size_t i, size_t j, size_t k) const;
size_t getActiveIndexLGR(const std::string& label, size_t localIndex) const;
```
Takes **0-based** i,j,k local coordinates. Returns `size_t` active index within LGR.

### `activeIndexLGR()` (line 106)
Lowercase variant — same signature, same semantics.

### `getTotalActiveLGR()` (line 102)
Total number of active cells across all LGRs.

### `get_lgr_cell_index()` (line 116)
Returns the cell index within `all_lgr_labels` for a given LGR tag.
Throws if LGR not found.

## Key Facts for PR3/PR4/PR6

| Fact | Detail |
|------|--------|
| `get_all_lgr_labels()` skip index 0 | Returns `labels[1..end]` — index 0 is global grid |
| `getActiveIndexLGR` coords | **0-based** i,j,k (NOT 1-based) |
| `keywordLB` design decision | Does NOT resolve active index at parse time — stores raw `lgr_.ijk`; evaluator calls `getActiveIndexLGR(lgr_name, i-1, j-1, k-1)` |
| `EclipseGrid` availability | Available in `Factory` as `this->grid_` — passed into constructor |
| `EclipseGrid` in `handleKW` | NOT available — only `GridDims` is passed. Confirmed in masterplan. |

## Key Structures
## LGR Identity API (key functions)

get_lgr_cell_index(lgr_tag: string) → size_t
  - Finds position of lgr_tag in get_all_lgr_labels() (0-based within that list)
  - all_lgr_labels[0]='GLOBAL' is excluded from get_all_lgr_labels() output
  - So LGR1 at all_lgr_labels[1] → get_lgr_cell_index returns 0
  - Used to convert LGR name → LGR number for SMSPEC NUMS array

get_lgr_labels_by_number(num: size_t) → string
  - Returns all_lgr_labels[num] directly (includes GLOBAL at 0)
  - Inverse of get_lgr_cell_index (+1 offset)

get_all_lgr_labels() → vector<string>
  - Returns all_lgr_labels[1..end] (skips GLOBAL at index 0)
  - Empty if no LGRs defined

all_lgr_labels: vector<string> (protected member)
  - [0] = 'GLOBAL' always
  - [1..N] = LGR names in declaration order

getActiveIndexLGR(label, i, j, k) → size_t
  - Resolves (LGR name, 1-based ijk) to active index within that LGR
  - Called by LB* evaluators to compute number_ at evaluation time

## Findings Log
| date | context | finding |
|------|---------|--------|
| 2026-05-06 |  | | date | context | finding |
|------|---------|---------|
| 2026-05-06 | PR0 review bska | Reviewer wants SummaryConfigNode to use optional<string> lgr_name + optional<size_t> lgr_number (LGR index from get_lgr_cell_index), NOT optional<lgr_info> with ijk. Reasoning: ijk→number_ conversion deferred to evaluator via getActiveIndexLGR; array::operator< produces undesirable lexicographic ordering on ijk tuples. |
| 2026-05-06 | design | get_lgr_cell_index(name) converts LGR name to 0-based index within get_all_lgr_labels(). Note offset: all_lgr_labels[0]=GLOBAL excluded, so LGR at position 1 → index 0 from get_lgr_cell_index. This is the number to store in SummaryConfigNode::lgr_number_. |
| 2026-05-06 | design | For SMSPEC: LGRS array = LGR name string (8-char padded), NUMS array = active cell index within LGR (computed by getActiveIndexLGR at write time, not stored in SummaryConfigNode). The lgr_number_ stored in SummaryConfigNode can serve as a stable sort key (monotonically increasing with LGR declaration order). | |
