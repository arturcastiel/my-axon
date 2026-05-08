# PR-001 — Technical Explanation

**Branch:** lgr-summaryconfig
**Depends on:** PR #5140 (merged, commit `0066645cd`)
**Blocks:** PR-002 (OutputStream SMSPEC arrays), PR-003 (LW* evaluators)

---

## Context

OPM supports Local Grid Refinement (LGR): fine-grained sub-grids embedded
inside a coarser global grid. Wells and connections can be declared inside
an LGR using `WELSPECL` / `COMPDATL` instead of the standard `WELSPECS` /
`COMPDAT`. The simulator produces LGR-specific summary vectors (`LW*`, `LC*`,
`LB*`) analogous to the global `W*`, `C*`, `B*` keywords, but scoped to a
named LGR grid.

Before this PR, opm-common silently discarded all `LW*`/`LC*`/`LB*` keywords
encountered in the SUMMARY section — the parser had no schema for them, and
`SummaryConfig` had no routing path for the `L` prefix.

## What this PR adds

This PR is the **config-layer half** of LGR summary vector support:

1. Parser schemas so the deck reader accepts `LW*`/`LC*`/`LB*`/`NOSUMLGR`.
2. `SummaryConfig` routing that converts each deck record into a
   `SummaryConfigNode` with `lgr_name_` set.
3. Unit and integration tests that verify the full parse-to-node path.

The **IO-layer half** (writing LGR metadata arrays to SMSPEC, evaluating
vectors at runtime) is handled by PR-002 and PR-003.

---

## Architecture: two-layer design

```
Deck keywords          SummaryConfig layer         IO / SMSPEC layer
─────────────          ───────────────────         ─────────────────
LWBHP 'LGR1' 'INJ'  → SummaryConfigNode           EclIO::SummaryNode
                        .keyword = "LWBHP"          .keyword = "LWBHP"
                        .category = Well            .category = Well
                        .lgr_name_ = "LGR1"  ────► .lgr = lgr_info {
                        .namedEntity = "INJ"              "LGR1", {i,j,k} }
```

`SummaryConfigNode` (this PR) stores only the **LGR name** — a string. It
knows nothing about cell coordinates inside the LGR. The conversion operator
in `SummaryConfig.hpp` bridges the two layers, producing
`lgr_info{"LGR1", {}}` with an empty `ijk` placeholder. The IO layer
(PR-002) fills in the actual cell coordinates from the SMSPEC
`NUMLX`/`NUMLY`/`NUMLZ` arrays.

This separation is intentional: the config layer deals with *what* to
request; the IO layer deals with *where* in the grid that is.

---

## Keyword schema design

### Why table format (no `"size"` field)?

A schema with `"size": 1` means the parser reads exactly one record and stops.
LW*/LC*/LB* keywords need to support multiple records (one per LGR/well
combination), so the `"size"` field must be absent — this tells the parser to
use table format and read until the block terminator `/`.

Early versions of `LOCAL_WELL_PROBE` had `"size": 1`, which caused Jenkins to
reject decks with more than one record per keyword with:
`"Extra line starting with '/' at line N"`. Removing `"size"` fixed this.

### Well-name expansion (`size_type: ALL`)

The `WELLS` item in `LOCAL_WELL_PROBE` uses `size_type: ALL`, which enables
wildcard pattern matching (e.g., `'*'` expands to all wells). This is
consistent with how the standard `WBHP` / `WOPR` keywords handle well
patterns.

---

## Implementation details

### `keywordLW` and `keywordLC` — well-tag filtering

LGR wells are declared with `WELSPECL` instead of `WELSPECS`. This stores the
LGR name inside the `Well` object as the *well tag* (`get_lgr_well_tag()`).

`keywordLW` iterates all wells matching the name pattern from the deck record,
then **filters** by LGR tag:

```cpp
for (const auto& wname : candidates) {
    const auto& well = schedule.getWellatEnd(wname);
    if (well.get_lgr_well_tag() != lgr_name) {
        continue;
    }
    list.push_back(param.namedEntity(wname).lgr_name(lgr_name));
}
```

This ensures that a wildcard pattern like `'*'` only picks up wells that
actually live in the specified LGR, not global-grid wells with the same name
pattern. `keywordLC` follows the same pattern for the `Connection` category.

### `keywordLB` — no well lookup

Block vectors (`LB*`) are not associated with a well — they address a cell
directly by LGR name and local (I, J, K) coordinates. `keywordLB` therefore
has no well-tag filtering: it builds one `Block` node per deck record, storing
only the LGR name. The cell coordinates from the deck are intentionally
discarded at this layer (see architecture section above).

### `NOSUMLGR`

`NOSUMLGR` is a processing instruction (no items) that, when present in the
SUMMARY section, signals the simulator to suppress LGR summary output. It is
handled via the existing `is_processing_instruction` /
`handleProcessingInstruction` mechanism already used for `SEPARATE`,
`NARROW`, etc. The flag is stored as `noSumLgr_` (bool) in `SummaryConfig`
and propagated via `serializeOp`.

---

## Test strategy

### `ParseKeywords/Category`

Verifies the `parseKeywordCategory` routing table change — that the new
`case 'L'` correctly maps `LWOPR` → `Well`, `LCOFR` → `Connection`,
`LBPR` → `Block`, and falls through to `Miscellaneous` for an unknown `LX*`
keyword. These are pure unit tests with no deck parsing.

### `NoSumLgr`

Two cases: `default_is_false` checks the default state without a deck;
`serialization_test_object_sets_flag` exercises the serialization roundtrip
for the `noSumLgr_` field.

### `LGR_SummaryConfig_Deck` — integration tests

Six tests against two `.DATA` files. These are the primary end-to-end tests:
they parse a real deck, build `SummaryConfig`, and assert on the resulting
node set.

Key assertions:
- LW* nodes carry `lgr_name_` set to the correct LGR.
- A global-grid well (`WOPR 'PROD'`) produces a node with **no** `lgr_name_`
  (regression guard — the routing must not leak LGR state into global vectors).
- Two LGR names for the same keyword produce two distinct nodes — the
  `operator==` / `operator<` changes from PR-000 are exercised here via the
  `std::set<SummaryConfigNode>` dedup path.

### `LGR_Schema_Inline` — schema regression tests

Five tests that construct a complete deck inline (no external files). They
exist to catch schema-format regressions (wrong `"size"`, missing items, bad
regex) before Jenkins. `WELSPECL` is used in the inline schedule to establish
the well tags — `WELSPECS` alone would cause the LGR well-tag filter to
silently produce zero nodes.

---

## What this PR does NOT do

- **No SMSPEC output**: LGR metadata arrays (`LGRS`, `NUMLX`, `NUMLY`,
  `NUMLZ`) are written by PR-002 (`OutputStream`).
- **No runtime evaluation**: `LW*` evaluator functions live in PR-003
  (`Summary.cpp`). Without PR-003, the summary config nodes are correctly
  built but no values are written to UNSMRY.
- **No `LB*` evaluators**: block-level LGR evaluation is PR-006
  (opm-simulators scope).
- **No ijk in SummaryConfigNode**: cell coordinates inside the LGR are an
  IO-layer concern. They are not stored here and are not needed for
  deduplication (two LB* nodes for the same LGR and same ijk are genuinely
  the same request).
