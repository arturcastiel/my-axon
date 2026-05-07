# Shadow: opm/io/eclipse/OutputStream.hpp
project: opm-common
indexed: 2026-05-05

## Summary
Header for the Eclipse binary output stream layer. Defines four stream manager
classes (Init, Restart, RFT, SummarySpecification) plus helper structs. Each
class wraps a `std::unique_ptr<EclOutput>` and exposes typed `write()` methods.
`SummarySpecification` is the target class for PR02.

## Structures / Classes

### `SummarySpecification` (line ~377)
Manages the SMSPEC file. Nested classes:
- `Parameters` — value accumulator; one call to `add()` per summary vector;
  holds 4 parallel `vector<PaddedOutputString<8>>` (keywords, wgnames, units)
  and one `vector<int>` (nums). Friend of SummarySpecification.
- `UnitConvention` enum (Metric/Field/Lab/Pvt_M)
- `RestartSpecification` struct (root + step)

Key methods:
- `Parameters::add(keyword, wgname, num, unit)` — the only public mutator;
  appends to all 4 vectors atomically.
- `write(params, simulationFinished, currentStep, basic)` — serialises
  SMSPEC arrays via `smspec.write(name, vector)` calls.
- Private: `rewindStream()`, `flushStream()`, `stream()`.

### Existing includes (line 24-31)
`PaddedOutputString.hpp`, `TimeService.hpp`, `<array>`, `<chrono>`, `<ios>`,
`<memory>`, `<string>`, `<vector>`.
**Missing for PR02**: `<optional>`, `<opm/io/eclipse/SummaryNode.hpp>`.

## Key Facts for PR02
| Fact | Detail |
|---|---|
| `Parameters` private section | `keywords`, `wgnames`, `units` are `vector<PaddedOutputString<8>>`; `nums` is `vector<int>` |
| No LGR fields | No `lgrs`, `numlx`, `numly`, `numlz` vectors exist |
| Single `add()` overload | 4-arg: keyword, wgname, num, unit |
| `write()` does NOT emit LGR arrays | No LGRS/NUMLX/NUMLY/NUMLZ/LGRNAMES |
| `SummaryNode.hpp` not included | Adding it introduces `lgr_info` to this TU; check for circular deps |
| No circular dep risk | `SummaryNode.hpp` includes only `<array>`, `<functional>`, `<limits>`, `<optional>`, `<string>` |

## PR02 Change Points
1. Add `#include <opm/io/eclipse/SummaryNode.hpp>` and `<optional>` to includes
2. Add `add()` overload declaration with 5th arg `const std::optional<Opm::EclIO::lgr_info>&`
3. Add 4 private vectors to `Parameters`: `lgrs` (PaddedOutputString<8>), `numlx`, `numly`, `numlz` (int)
