# High-Level Plan — axon-3.0
Updated: 2026-05-08  ·  Phase 2 — Plan based on Phase 1 study  ·  20 PRs across 4 phases

## Context (from Phase 1)

**Goal**: Bring AXON from "observe-only" to "self-improving". The infrastructure for autonomy is already 80% built — every signal store exists, every tool has its place, every gate is wired. What's missing is the orchestrator that closes loops without per-step user confirmation, the audit trail that makes auto-edits accountable, and a small batch of foundation fixes that eliminate today's silent breakage (cron not firing, session snapshot empty, drift score read by nothing).

The headline question — *can AXON improve itself automatically based on use?* — is answerable as "yes, with the work below". This plan turns that into a buildable sequence.

## Architecture overview

The plan respects AXON's existing layer boundaries:

```
Layer 1 — Kernel (axon/)         ← Phase A modifies via dev-mode (compiler grammar, KERNEL-SLIM)
Layer 2 — Workspace config       ← Phase A/B/C/D writes programs, preferences, scheduler entries
Layer 3 — Tools                  ← Phase A/B refactors tools, adds probes, consolidates
Layer 4 — User data (my-axon)    ← Phase C/D writes audit logs, rollback snapshots, orchestrator state
```

Self-improvement uses the existing closed-loop primitives (igap, usage, drift, dispatch-feedback, pattern, cron, events). Nothing new is invented at the conceptual level — Phase 2 just connects the wires.

---

## Phase A — Foundation (PR-001 to PR-005)

Make the system honest about its state before adding autonomy. Fix the silently-broken loops, normalize tool conventions, and complete partially-specified subsystems.

**Theme**: Reliability + completeness. Every PR here is a small/medium fix that unblocks Phase B/C/D.

- **PR-001 — Foundation fixes**: cron.py:165 TypeError, session-snapshot empty, stale `KERNEL.md` references in HOWTO.md, version-stamp drift between KERNEL-SLIM (v1.1.0) and DEVELOPER.md changelog (v1.1.1). All bite-sized, all blocking.
- **PR-002 — Tool authoring conventions**: standardize argparse to subparsers across the suite, define canonical JSON error envelope (`{"ok": bool, "error": str|null, "data": ...}`), introduce atomic-write helper for kv stores (`memory.py`, `events.py`).
- **PR-003 — Path adoption + probe coverage**: migrate the 4 remaining holdouts (`shadow`, `document_parser`, `web_search`, `test_runner`) to `_axon_paths`; add functional probes for 6 critical kernel tools that are smoke-only today (compile-write, dispatch, usage, events, igap, axon-audit).
- **PR-004 — Library-dev compilation**: compile the 9 uncompiled library-dev programs + the metas missing from compiled/ (igap-improve, code-dev-pr-review, code-dev-shadow). Closes the visible 0/9 coverage gap and gives Phase D's auto-compile loop a clean starting state.
- **PR-005 — Compiler completeness**: fill compiler/GRAMMAR.md with rules for EVAL / RETRY / TEE (defined in LANG v2.3.0 but not in grammar). Complete the truncated COMPILER.md Phase 3 (OPTIMIZE) spec. Required so the compiler stops emitting warnings on these ops and so user-authored programs can use them in natural language.

## Phase B — Tool consolidation (PR-006 to PR-008)

Reduce surface area before adding autonomy on top.

**Theme**: Fewer tools, clearer pipeline, no behavior change.

- **PR-006 — events + hooks unification**: events emit; hooks listen. Today they're separate tools with overlapping config. Merge into one `events` tool with `--listen`/`--emit`/`--clear` subcommands. Saves 1 tool, makes the event-driven story coherent.
- **PR-007 — compile pipeline consolidation**: `compile-write` + `compile-suggest` + `compile-optimizer` → single `compile` tool with subcommands (`format`, `rank`, `optimize`, `verify`, `test-all`). Keeps internal modules but presents one CLI. Saves 2 tools and removes the subprocess hop between compile-suggest and compile-write.
- **PR-008 — Smaller merges**: `save-workspace` ⊕ `workspace-backup` (both push my-axon to remote). `code-dev-pr` ⊕ `code-dev-pr-spec` with `--style {default,opm,detailed}` flag. Surfaces clear ownership of each capability.

## Phase C — Self-improvement plumbing (PR-009 to PR-016)

Activate the loops one at a time, each with its own gate.

**Theme**: Wire the existing observability stores into actual decisions, with audit + rollback as table stakes.

- **PR-009 — Session-save handoff repair**: the empty `last-session-snapshot.md` bug, but expanded to actually capture the W: keys needed for resume. Adds a pytest case that round-trips W: state through save→load. Required so future sessions can be picked up by the orchestrator.
- **PR-010 — Cron daemon activation**: today the cron infrastructure exists but no scheduler fires entries (`run_count=0` for the only registered job since 2026-05-03). Add a boot-time tick that surfaces overdue cron entries and either auto-runs them (when L:cron-auto ≡ true) or queues them with a one-line user prompt.
- **PR-011 — Default self-improvement cron schedule**: register the 5 weekly/monthly self-care jobs the system was clearly designed for: `igap report --days 7`, `compile-suggest suggest --window 7d`, `dispatch-stats weekly`, `memory-compact --age 30d`, `session-save` daily.
- **PR-012 — Drift gating**: today drift score is computed but read by nothing. Wire it: high drift (>0.6) → halt with FAIL block; medium drift (0.3–0.6) → confidence-modifier in output layer; low → quiet. Adds drift to the response gate's input vector.
- **PR-013 — Prompt-log opt-in flow**: prompt-log is hard-disabled today (`prompt-log-enabled.md = false`). Add a one-time consent program that explains what gets logged, where, and how to disable. Once enabled, the pattern → compile loop becomes feedable.
- **PR-014 — Dispatch auto-tune**: the threshold-tuning code already exists in `dispatch.py:214-236` but the feedback log is empty (no integration recorded feedback yet). Wire feedback recording into the dispatch flow: after a match, the next user input either confirms or doesn't. Heuristic: `igap absent-instruction within 2 turns of a dispatch ≡ negative feedback`.
- **PR-015 — Audit trail for auto-edits**: every auto-applied change (compiled program rewrite, threshold adjustment, kernel patch via igap-improve) records to `my-axon/log/auto-edits/YYYY-MM-DD.md` with: program, timestamp, rule applied, before/after diff, gate that allowed it. Daily `auto-actions` summary program reads it and surfaces a recap on next boot.
- **PR-016 — Rollback for auto-edits**: extend `memory.py`'s 3-version L: rollback pattern to (a) compiled programs (`workspace/programs/compiled/.rollback/`) and (b) W: keys before each auto-edit. Single `undo` program lookup point. Makes auto-improvement reversible and therefore safe enough to enable.

## Phase D — The orchestrator + UX (PR-017 to PR-020)

Close the loop. Once Phase A/B/C lands, this is where AXON starts driving itself.

**Theme**: Autonomous self-care, with full accountability surface.

- **PR-017 — Auto-improvement orchestrator** (this is the headline PR): a new program that runs daily (cron from PR-011), reads the signal stores, identifies actions, and applies the safe ones automatically. Gated by: (a) `L:auto-improve ≡ true` (defaults false, opt-in), (b) drift ≤ medium, (c) dev-mode for any kernel-affecting fix, (d) every action records to PR-015's audit trail and is rollback-able via PR-016. Specific rules: *auto-compile* programs ≥5 uses, *auto-tune* dispatch threshold on neg-rate, *auto-archive* episodic memory >30d. Anything beyond compiled programs (kernel patches, LANG extensions, new tools) stays human-confirmed.
- **PR-018 — Menu + output-layer UX**: surface meta tools (axon-audit, compile-optimizer, axon-docs-gen, auto-actions) on the menu. Add a "Self-improvement" panel showing: drift state, last cron run, pending auto-actions, last 3 auto-edits with link to audit trail. Drift score becomes visible — closes the loop on PR-012.
- **PR-019 — Inter-tool shared library**: today 5+ tools shell out to other tools as subprocesses, duplicating arg parsing and slowing the orchestrator. Refactor to importable functions in a `tools/_axon_lib.py` (or similar) where the call hops are hottest. No CLI change.
- **PR-020 — Programs in REGISTRY**: extend `tools/REGISTRY.json` schema to optionally include programs (or split into `REGISTRY-PROGRAMS.json`). Today every program's metadata lives in its own header — fine for humans, expensive for the orchestrator to scan. Single registry → one read on boot.

---

## Risks + non-goals

**Risks**:
- **PR-017** is the riskiest — autonomous edits to anything inside the repo. Mitigation: only auto-applies *compile* (regenerable, rollback-able), *tune* (single-value), *archive* (no data loss); kernel/LANG/program *content* changes stay human-confirmed.
- **PR-013** could be perceived as surveillance — mitigation: explicit consent, local-only storage, easy disable, no opt-in default.
- **PR-010** can wake up a stale system noisily on first boot after a long absence — mitigation: rate-limit catchup (run at most one overdue per boot, queue the rest).

**Non-goals (intentionally out of scope)**:
- Cross-machine federation (my-axon scope assumes one user/host).
- Multi-agent shared workspace.
- Auto-extending LANG.md (Core Rule 10: EXTEND protocol stays manual).
- Auto-editing kernel files (Core Rule 9: write gate keeps `axon/` human-only).
- Replacing manual `igap improve` with full auto-fix of inferred gaps — too risky; orchestrator only auto-applies the 3 narrow categories above.

## Assumptions

- AXON stays single-user, single-host.
- The git remote for axon.git stays open for push (CI is not a required gate; pre-commit + manual review hold).
- Health-score remains a hard requirement of 100 in the dev loop (locally).
- Dev-mode toggle is the human's discretionary tool; orchestrator never enables it.

## Sequencing rationale

A is gated to land first because broken cron and broken handoff make C/D meaningless. B is independent of C/D but depends on A's tool conventions; sandwiched between for context-pressure reasons (cleaner tool surface = simpler orchestrator). C is the long stretch — each PR closes one loop. D ties them.

Estimated effort: A ≈ 2 dev-days. B ≈ 2 dev-days. C ≈ 4 dev-days. D ≈ 3 dev-days. **Total: ~11 dev-days**, splittable.

## What "done" looks like

- Health 100/100 from any cwd, on any machine.
- Lint clean on every commit (already true post-Q1 work).
- `cron.json` shows `run_count > 0` for at least one self-improvement entry.
- `my-axon/log/auto-edits/` has at least one recorded auto-action that ran without user prompt.
- `last-session-snapshot.md` is non-empty after a real session.
- `discover` reports zero high-drift programs.
- Menu shows the self-improvement panel.
