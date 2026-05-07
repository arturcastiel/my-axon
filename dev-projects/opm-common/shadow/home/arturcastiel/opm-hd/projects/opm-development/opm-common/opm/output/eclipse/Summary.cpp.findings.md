# Shadow: opm/output/eclipse/Summary.cpp
project: opm-common
indexed: 2026-05-06
git-hash: (Study 5 — pre-PR3)
lines: 5971

## Summary
The evaluator layer — computes summary vector values at each report step and
writes them to UNSMRY. Contains all `Base`-derived evaluator classes, the
`Factory::create()` dispatch chain, `FunctionRelation` orchestration, and the
outer `Summary` class that drives the write loop.

**LGR status:** All LGR nodes (`LWOPR`, `LCOFR`, `LBPR`, etc.) currently fall
through `Factory::create()` to `unknownParameter()` — they are silently ignored.
PR3 (LW* well evaluators) and PR4 (LC* connection evaluators) add new evaluator
classes and a new `isLgrWellValue()` / `isLgrConnectionValue()` guard before
`isFunctionRelation()`.

---

## Key Architecture

### `Base` class (line 4104)
```cpp
class Base {
public:
    virtual ~Base() {}
    virtual void update(const std::size_t sim_step, const double stepSize,
                        const InputData& input, const SimulatorResults& simRes,
                        Opm::SummaryState& st) const = 0;
};
```
All evaluators inherit from `Base`. `update()` is called once per report step.

### `InputData` struct (line ~4090)
Available inside every `update()` call:
- `input.sched` — `Opm::Schedule`
- `input.es` — `Opm::EclipseState`
- `input.grid` — `Opm::EclipseGrid`
- `input.reg` — `Opm::RegionCache`
- `input.initial_inplace` — `Opm::Inplace`

### `SimulatorResults` struct (line ~4095)
- `simRes.wellSol` — `Opm::data::WellRates` (keyed by well name)
- `simRes.grpNwrkSol` — group network solution
- `simRes.inplace` — `Opm::Inplace`

### `FunctionRelation` evaluator (line 4116)
The main well/connection evaluator. Uses `ofun` (function pointer into `funs` map).
Flow:
1. `find_wells()` — resolves well names from schedule for this step
2. `EfficiencyFactor::setFactors()` — computes efficiency
3. Builds `fn_args` struct with all data
4. Calls `this->fcn_(args)` → returns `{unit, value}`
5. `updateValue(node, converted_value, st)` — writes to SummaryState

### `Factory::create()` dispatch chain (line 4720)
```
isUserDefined()       → userDefinedValue()
isBlockValue()        → blockValue()         ← uses block_units map + cellActive()
isAquiferValue()      → aquiferValue()
isRegionValue()       → regionValue()
isInterRegionValue()  → interRegionValue()
isGlobalProcessValue()→ globalProcessValue()
isFunctionRelation()  → functionRelation()   ← uses funs map
                      → unknownParameter()   ← LGR nodes land here (broken)
```

**PR3 insertion point:** Add `isLgrWellValue()` check BEFORE `isFunctionRelation()`.
LGR nodes have `node.lgr.has_value() == true` — use this as the guard.

**PR4 insertion point:** Add `isLgrConnectionValue()` check alongside PR3's check.

---

## Key Facts for PR3/PR4

| Fact | Detail |
|------|--------|
| Insertion point | After `isGlobalProcessValue()`, before `isFunctionRelation()` in `Factory::create()` (line ~4737) |
| LGR node guard | `this->node_->lgr.has_value()` — true for all LW*/LC*/LB* nodes |
| `funs` map | Does NOT contain LWOPR, LCOFR, etc. — `isFunctionRelation()` returns false for LGR nodes |
| `block_units` map | Does NOT contain LB* keywords — `isBlockValue()` returns false |
| `isBlockValue()` uses | `this->grid_.cellActive(this->node_->number - 1)` — LGR nodes have `number=0`, would underflow |
| `SummaryState::update(key, val)` | `key = node.unique_key()` — includes LGR name in key |
| `updateValue()` helper | Calls `st.update(node.unique_key(), value)` |
| `find_wells()` | Returns `vector<const Well*>` matching `node.wgname` from schedule at `sim_step` |
| `fn_args` struct | Large struct — LGR evaluator should NOT use it; extract values directly from `simRes.wellSol` |
| Well data access | `simRes.wellSol.at(well_name)` → `Opm::data::Well` → `.rates.get(rt::oil)` etc. |
| Connection data | `simRes.wellSol.at(well_name).connections` → `vector<Opm::data::Connection>` |

## `ofun` type (line ~2549)
```cpp
using ofun = std::function<quantity(const fn_args&)>;
```
`funs` is `std::unordered_map<std::string, ofun>`. Does not contain LGR keywords.

## LGR Well Evaluator Pattern (for PR3)
New class `LgrWellValue : public Base`:
- Constructor: takes `SummaryNode node`, `UnitSystem::measure m`, `rt rate_type`
- `update()`: look up `simRes.wellSol[node_.wgname]`, read `.rates.get(rate_type)`, convert units, call `updateValue()`
- Guard in Factory: `node->lgr.has_value() && node->category == Category::Well`

## LGR Connection Evaluator Pattern (for PR4)
New class `LgrConnectionValue : public Base`:
- Constructor: takes `SummaryNode node`, `UnitSystem::measure m`, `rt rate_type`
- `update()`: look up `simRes.wellSol[wgname]`, find connection matching `node_.lgr->ijk`, read rate
- Guard in Factory: `node->lgr.has_value() && node->category == Category::Connection`

## Includes already present
`#include <opm/input/eclipse/Schedule/Well/Well.hpp>` — line 55 ✅
`#include <opm/output/data/Wells.hpp>` — line 75 ✅
All LGR evaluator code can use these without new includes.
