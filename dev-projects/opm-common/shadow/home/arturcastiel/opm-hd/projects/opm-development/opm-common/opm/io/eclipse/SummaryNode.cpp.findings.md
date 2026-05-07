# SHADOW: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/io/eclipse/SummaryNode.cpp
source-path: /home/arturcastiel/opm-hd/projects/opm-development/opm-common/opm/io/eclipse/SummaryNode.cpp
shadow-created: 2026-05-05
shadow-updated: 2026-05-05
git-hash: e6e5cca3732006ac22217c91e7631cf1d4b19566126154cc38ab21c45df9704d

## Summary
Implementation of `SummaryNode` methods. Key function: `category_from_keyword()` — routes a summary keyword string to its `Category` enum value by inspecting the first (and sometimes second) character. PR0 added `case 'L':` with a nested switch on `keyword[1]` to handle LW*, LC*, LB* prefixes.

## Key Structures

### `category_from_keyword()` — the classification engine
```cpp
switch (keyword[0]) {
case 'A': return Category::Aquifer;
case 'B': return Category::Block;
case 'C': return distinguish_connection_from_completion(keyword);
case 'F': return Category::Field;
case 'G': return distinguish_group_from_node(keyword);
case 'L':                                          // PR0 ADDITION
    if (keyword.size() >= 2) {
        switch (keyword[1]) {
        case 'W': return distinguish_well_from_completion(keyword);  // LW* → Well
        case 'C': return distinguish_connection_from_completion(keyword); // LC* → Connection
        case 'B': return Category::Block;           // LB* → Block
        default:  break;
        }
    }
    return Category::Miscellaneous;
case 'R': return Category::Region;
case 'S': return Category::Segment;
case 'W': return distinguish_well_from_completion(keyword);
default:  return Category::Miscellaneous;
}
```

### Helper predicates (file-local, anonymous namespace)
- `use_number(category)` → bool — Block, Connection, Completion, Region, Segment, Aquifer return true
- `use_name(category)` → bool — Well, Group, Connection, Completion, Segment, Node return true
- `is_node_keyword(kw)` — only "GPR" currently
- `is_connection_completion(kw)` — regex `C[OGW][IP][RT]L`
- `is_well_completion(kw)` — regex `W[OGWLV][PIGOLCF][RT]L([0-9_]{2}[0-9])?`
- `distinguish_group_from_node(kw)` — Group unless is_node_keyword
- `distinguish_connection_from_completion(kw)` — Connection vs Completion
- `distinguish_well_from_completion(kw)` — Well vs Completion
- `normalise_well_completion_keyword(kw)` — strips numeric suffix from WOPRL123 → WOPRL

### `unique_key()` construction
Format: `keyword[:name][:number]` — uses `display_name()` (if `use_name`) and `display_number()` (if `use_number`). Delimiter is `':'`.

## Dependencies
- `<opm/io/eclipse/SummaryNode.hpp>` — self header
- `<numeric>`, `<regex>`, `<string>`, `<unordered_set>`, `<vector>`

## Architecture Role
The classification decision point for the entire summary pipeline. The `case 'L':` block added in PR0 is what causes LGR keywords to reach the correct category — without it, all LW*/LC*/LB* keywords fall through to `Miscellaneous`, breaking the entire LGR summary feature.

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR0 | Added `case 'L':` between `case 'G':` and `case 'R':` in `category_from_keyword()` |
| 2026-05-05 | PR0 | LW* → delegates to `distinguish_well_from_completion()` (handles LWOPRL completion suffix) |
| 2026-05-05 | PR0 | LC* → delegates to `distinguish_connection_from_completion()` (handles LCOPRL completion) |
| 2026-05-05 | PR0 | LB* → direct `Category::Block` (no completion variant for block keywords) |
| 2026-05-05 | PR0 | Single-char "L" alone → falls through `default: break` → returns `Miscellaneous` |
| 2026-05-05 | design | `distinguish_well_from_completion(kw)` reused unchanged — the regex matches `LW*` correctly because it matches `[OGWLV][PIGOLCF][RT]L` which starts at position 1 after the initial char, but wait — the regex is `W[OGWLV]...` starting with 'W'. For LW keywords, `keyword[0]='L'` so the keyword starts with L. The `distinguish_well_from_completion` function calls `is_well_completion(keyword)` which uses regex `W[OGWLV][PIGOLCF][RT]L...` — this requires keyword[0]='W'. For LWOPRL: keyword[0]='L', so regex does NOT match → returns Well (not Completion). This is correct for LGR well keywords. |
