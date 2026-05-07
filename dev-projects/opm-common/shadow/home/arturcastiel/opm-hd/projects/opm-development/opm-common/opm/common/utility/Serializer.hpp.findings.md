# Shadow: opm/common/utility/Serializer.hpp
project: opm-common
indexed: 2026-05-06

## Summary
OPM's serializer — pack/unpack objects for MPI and restart. Uses `operator()(T)`
which dispatches based on type traits.

## Dispatch order in `operator()(const T& data)` (line 104)
```
1. has_serializeOp<T>  → calls data.serializeOp(*this)
2. is_optional<T>      → calls optional(data)
3. is_array<T>         → calls array(data)  (std::array<T,N>)
4. else                → calls m_packer.pack/unpack(data)
   (handles: bool, int, double, std::string, std::vector<T>, ...)
```

## Key Facts for lgr_info serialization

| Type | How it serializes |
|------|------------------|
| `std::string` | packer handles directly (line 130-134) |
| `std::array<int,3>` | `is_array<T>` branch → `array()` helper (line 122-124) |
| `std::optional<lgr_info>` | `is_optional<T>` branch → `optional()` helper (line 116) |
| `lgr_info` inside optional | `has_serializeOp<lgr_info>` → calls `lgr_info::serializeOp(*this)` |

**PR0 design:** `lgr_info` needs `serializeOp` to pack `name` (std::string) and
`ijk` (std::array<int,3>). Without it, `optional<lgr_info>` serialization compiles
but silently fails to pack the contained value.

```cpp
// Correct lgr_info::serializeOp (added in PR0):
template<class Serializer>
void serializeOp(Serializer& s) {
    s(name);
    s(ijk);
}
```
