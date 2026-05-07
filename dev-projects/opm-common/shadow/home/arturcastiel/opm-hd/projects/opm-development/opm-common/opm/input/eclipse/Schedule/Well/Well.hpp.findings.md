# Shadow: opm/input/eclipse/Schedule/Well/Well.hpp
project: opm-common
indexed: 2026-05-06

## Summary
The `Well` class: represents a single well across all schedule steps. Contains
well connections, segments, LGR association, injection/production controls.

## LGR-relevant API

### `get_lgr_well_tag()` (line 116)
```cpp
std::optional<std::string> get_lgr_well_tag() const;
```
Returns the LGR name this well belongs to, or `std::nullopt` if it's a global well.
Set by `WELSPECL` keyword via `set_lgr_well_tag(lgr_tag_name)`.

### `set_lgr_well_tag()` (line 108)
```cpp
void set_lgr_well_tag(const std::string& lgr_tag_name);
```
Called by the `WELSPECL` handler in the schedule parser.

### Private field (line 709)
```cpp
std::string lgr_tag{};
```
Empty string = global well. Non-empty = LGR well tag.

### Serialization (line 641)
```cpp
serializer(lgr_tag);
```
`lgr_tag` is serialized. Restart files preserve LGR well association.

### Implementation (Well.cpp line 649)
```cpp
std::optional<std::string> Well::get_lgr_well_tag(void) const {
    if (lgr_tag.empty()) return std::nullopt;
    return lgr_tag;
}
```

## Key Facts for PR3 (LW* evaluators)

| Fact | Detail |
|------|--------|
| Filter pattern | `well.get_lgr_well_tag() == node.lgr->name` — match well to LGR in evaluator |
| `WELSPECS` wells | `lgr_tag` is empty → `get_lgr_well_tag()` returns nullopt — NOT LGR wells |
| `WELSPECL` wells | `lgr_tag` is set → `get_lgr_well_tag()` returns the LGR name — IS LGR well |
| Access pattern | `input.sched.getWellatEnd(well_name).get_lgr_well_tag()` |
| `sched.wellNames()` | Returns ALL wells (global + LGR). Filter by `get_lgr_well_tag()` in evaluator. |
