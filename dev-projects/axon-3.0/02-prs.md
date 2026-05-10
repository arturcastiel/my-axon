# PR List — axon-3.0
Updated: 2026-05-08  ·  Total PRs: 20  ·  Phase A (5) + Phase B (3) + Phase C (8) + Phase D (4)

---

## PR-001 — Foundation fixes
- **Status:** not-started
- **Complexity:** S
- **Phase:** A — Foundation
- **Scope:** `tools/cron.py` (line 165 TypeError on `data["jobs"]`), `tools/session_save.py` (snapshot writes empty file), `axon/HOWTO.md` (stale `KERNEL.md` references → KERNEL-SLIM.md), `axon/KERNEL-SLIM.md` line 2 (version stamp v1.1.0 → v1.1.1 to sync with DEVELOPER.md changelog)
- **Depends on:** none
- **Why:** Four bite-sized bugs blocking other work. Cron status is unreadable from CLI; session resume restores no state; HOWTO points at a file that was archived 6 days ago; version stamps disagree. None survive a CI gate but none are caught by it either.
- **Spec:** 03-prs/PR-001.md (not written yet)

## PR-002 — Tool authoring conventions
- **Status:** not-started
- **Complexity:** M
- **Phase:** A — Foundation
- **Scope:** Standardize argparse style across tools that mix subparsers and positional `action=` args (queue_tool.py, dispatch.py, compile_suggest.py, etc.). Define `tools/_axon_response.py` with canonical `ok()`/`fail()` helpers returning the JSON envelope `{"ok": bool, "error": str|null, "data": any}`. Add `tools/_axon_io.py` with atomic-write helper (temp + rename), apply to memory.py + events.py. Update tests/test_tools_kernel.py to assert envelope shape.
- **Depends on:** none
- **Why:** Today's tools have inconsistent JSON shapes, mixed stderr/stdout for errors, and direct file writes that race. Phase B's consolidation and Phase C's orchestrator both need a stable contract; without this, every consumer reinvents error parsing.
- **Spec:** 03-prs/PR-002.md (not written yet)

## PR-003 — Path adoption + probe coverage
- **Status:** not-started
- **Complexity:** M
- **Phase:** A — Foundation
- **Scope:** Migrate 4 holdout tools (`shadow.py`, `document_parser.py`, `web_search.py`, `test_runner.py`) to import from `_axon_paths`. Add functional probes in `health.py:PROBES` for: compile-write (format a known-good test program, parse output), dispatch (match a fixture query, assert program returned), usage (record→read roundtrip), events (emit→listen roundtrip), igap (record→report roundtrip), axon-audit (run with `--dry-run` and assert structural keys present). Each probe is ≤20 lines.
- **Depends on:** PR-002 (probe assertions use the canonical envelope)
- **Why:** Smoke-only probes don't catch regressions; today 36/49 ACTIVE tools are smoke-only including the ones the orchestrator depends on. Without functional probes, Phase C/D ride on hope.
- **Spec:** 03-prs/PR-003.md (not written yet)

## PR-004 — Library-dev compilation
- **Status:** not-started
- **Complexity:** M
- **Phase:** A — Foundation
- **Scope:** Run `compile-optimizer compile` on the 9 uncompiled library-dev programs (library-dev, library-dev-new, library-dev-ingest, library-dev-explain, library-dev-intersect, library-dev-search, library-dev-report, library-dev-cite, library-dev-status) and on the 3 uncompiled metas (igap-improve, code-dev-pr-review, code-dev-shadow). Verify each via `compile-optimizer test-all`. Update workspace/memory/longterm/dispatch-index.json. Add a regression test that fails on any program with a `# PROGRAM:` header but no `compiled/*.cmp.md`.
- **Depends on:** PR-005 (compiler grammar must accept EVAL/RETRY/TEE so library-dev's `intersect` and `report` compile cleanly)
- **Why:** Library-dev is the single biggest coverage gap (0/9 vs code-dev's 10/11). Phase D's auto-compile rule needs a clean baseline. The regression test makes the gap permanent-fixed.
- **Spec:** 03-prs/PR-004.md (not written yet)

## PR-005 — Compiler completeness
- **Status:** not-started
- **Complexity:** M
- **Phase:** A — Foundation
- **Scope:** `axon/compiler/GRAMMAR.md` — add natural-language → op rules for EVAL (e.g. "evaluate output against criteria" → `EVAL(output, criteria)`), RETRY ("retry up to N times" → `RETRY(op, condition, max=N)`), TEE ("save verbose output and summarize" → `TEE(result, key, summary?)`). `axon/compiler/COMPILER.md` — complete Phase 3 (OPTIMIZE) spec which currently truncates at line 99: define optimization passes (dead-store elimination, redundant RETRIEVE collapsing, EXEC inlining for trivial programs). Requires `L:dev-mode ≡ true` for both edits.
- **Depends on:** none (but sequenced before PR-004 since library-dev programs use these ops)
- **Why:** LANG defines 14 extensions but compiler grammar only maps the original 11. EVAL/RETRY/TEE programs today fall back to literal `EXEC("[description]")`, losing optimization. Half-spec compiler is also a maintenance trap.
- **Spec:** 03-prs/PR-005.md (not written yet)

## PR-006 — events + hooks unification
- **Status:** not-started
- **Complexity:** M
- **Phase:** B — Tool consolidation
- **Scope:** Merge `tools/hooks.py` into `tools/events.py` as `events emit`/`events listen`/`events hook --add`/`events hook --remove`/`events hook --list`. Remove `hooks.py` from `tools/REGISTRY.json` (keep alias in `axon.py:ALIASES` for one release). Update programs that call hooks (search workspace/programs/ for TOOL(hooks)). Migrate any existing hook configs from `workspace/memory/longterm/hooks.json` into the events tool's namespace.
- **Depends on:** PR-002 (canonical envelope, atomic writes)
- **Why:** Conceptually one system (event bus + listeners). Two tools with overlapping configs invites drift. Saves one tool, simplifies orchestrator's signal-routing.
- **Spec:** 03-prs/PR-006.md (not written yet)

## PR-007 — Compile pipeline consolidation
- **Status:** not-started
- **Complexity:** M
- **Phase:** B — Tool consolidation
- **Scope:** Single `tools/compile.py` with subcommands: `format` (was compile-write), `rank` (was compile-suggest), `optimize` + `verify` + `test-all` (was compile-optimizer). Keep internal modules (rename to `_compile_format.py`, `_compile_rank.py`, `_compile_optimize.py`) so health probes stay functional. Update `axon.py:ALIASES`. Update programs that invoke compile-* (workspace/programs/suggest-compile.md, axon-audit.md). Saves 2 tools, removes the inter-tool subprocess hop in compile-suggest's auto-compile path.
- **Depends on:** PR-002 + PR-005
- **Why:** Three tools for three phases of one pipeline. Subcommands match user mental model and remove subprocess hops that today serialize JSON between tools.
- **Spec:** 03-prs/PR-007.md (not written yet)

## PR-008 — Smaller merges
- **Status:** not-started
- **Complexity:** S
- **Phase:** B — Tool consolidation
- **Scope:** (a) `save-workspace` + `workspace-backup` → keep `workspace-backup` (more general; setup/push/status). Add `save-workspace` as alias. Update menu hint. (b) `code-dev-pr` + `code-dev-pr-spec` → keep `code-dev-pr` with `--style {default,opm,detailed}`. Move OPM-specific spec template into `workspace/templates/code-dev-pr-opm.tpl.md`. Update code-dev dispatcher.
- **Depends on:** none
- **Why:** Two pairs of programs that do the same thing with different presets. Disambiguates the user-facing surface.
- **Spec:** 03-prs/PR-008.md (not written yet)

## PR-009 — Session-save handoff repair
- **Status:** not-started
- **Complexity:** M
- **Phase:** C — Self-improvement plumbing
- **Scope:** `tools/session_save.py` — actually write W: keys to `last-session-snapshot.md` (today produces 0-line file). Schema: JSON dict, `{"ts": iso, "keys": {key: value}, "active_phase": str|null, "active_program": str|null}`. Cap value size at 2KB per key (skip + log if larger). `tools/boot.py` — read snapshot on resume, restore W: keys, surface "resumed N keys from {ts}". Pytest round-trip: save state, clear, boot, assert restored.
- **Depends on:** PR-001 (the empty-snapshot bug fix is part of PR-001; this PR adds the actual restore path)
- **Why:** Today resume offers continue/restart/skip but the underlying W: state is empty. Either fix it or remove the resume prompt. Phase D's orchestrator needs reliable handoff to pick up mid-flight work.
- **Spec:** 03-prs/PR-009.md (not written yet)

## PR-010 — Cron daemon activation
- **Status:** not-started
- **Complexity:** M
- **Phase:** C — Self-improvement plumbing
- **Scope:** Boot-time tick: after kernel internalization, `tools/boot.py` calls `cron check` (now working post-PR-001). For each overdue entry: if `L:cron-auto ≡ true` (default false), schedule it on the queue; else surface "N overdue cron jobs" with a one-line approve prompt. Add rate limit: at most 1 overdue auto-run per boot (rest stay queued for next boots). Update `tools/cron.py` to record `last_run_ts` + `run_count` correctly. Test: register a cron, advance fake clock, boot, assert ran.
- **Depends on:** PR-001
- **Why:** Cron has been registered since 2026-05-03 with `run_count=0`. Without an active scheduler, nothing in Phase C/D autonomy plan ever fires. This is the smallest change that unblocks the whole self-improvement story.
- **Spec:** 03-prs/PR-010.md (not written yet)

## PR-011 — Default self-improvement cron schedule
- **Status:** not-started
- **Complexity:** S
- **Phase:** C — Self-improvement plumbing
- **Scope:** Add 5 default cron entries on first boot of v3.1 (only if absent — never overwrite user entries): `igap report --days 7` weekly Monday 09:00, `compile-suggest suggest --window 7d` weekly Monday 09:05, `dispatch-stats weekly` weekly Monday 09:10, `memory-compact --age 30d` monthly first-of-month 09:00, `session-save` daily 23:00. Idempotent registration via `cron add --if-absent`.
- **Depends on:** PR-010
- **Why:** Without these defaults, PR-010's daemon has nothing to run. The 5 jobs match exactly what the system was clearly designed for (their tools exist; their consumers are wired; they were just never scheduled).
- **Spec:** 03-prs/PR-011.md (not written yet)

## PR-012 — Drift gating
- **Status:** not-started
- **Complexity:** M
- **Phase:** C — Self-improvement plumbing
- **Scope:** Wire `tools/drift.py check` into the response gate. New rule: drift > 0.6 → HALT with FAIL block citing the divergent program; drift 0.3–0.6 → reduce W:response-confidence by 30% (cumulative with existing confidence calc); drift < 0.3 → quiet. Add drift status to `output-layer.md` footer (already partially done — now reads vs writes). Add menu badge "drift: stable|drifting|diverged". Pytest: synthesize a divergent program, run, assert HALT.
- **Depends on:** PR-002
- **Why:** Drift is computed every turn, read by zero decision logic. Either it matters (and should affect behavior) or it should be removed. This PR makes it matter.
- **Spec:** 03-prs/PR-012.md (not written yet)

## PR-013 — Prompt-log opt-in flow
- **Status:** not-started
- **Complexity:** S
- **Phase:** C — Self-improvement plumbing
- **Scope:** New program `workspace/programs/prompt-log-consent.md`. Explains: what's logged (verbatim user prompts), where (`my-axon/memory/episodic/prompt-log.jsonl`, gitignored), retention (90 days, auto-purged by memory-compact), how to disable (`prompt-log disable`), how to delete history (`prompt-log purge`). Sets `L:prompt-log-enabled ≡ true` only after explicit yes. Surface as a once-per-install prompt on menu when `L:prompt-log-consent-asked ≢ true` AND `L:prompt-log-enabled ≢ true`.
- **Depends on:** none
- **Why:** Today prompt-log is hard-disabled, which gates the entire pattern → compile loop. Auto-enabling is wrong; never offering is also wrong. One-time clear consent + easy off-switch.
- **Spec:** 03-prs/PR-013.md (not written yet)

## PR-014 — Dispatch auto-tune
- **Status:** not-started
- **Complexity:** M
- **Phase:** C — Self-improvement plumbing
- **Scope:** `tools/dispatch.py` — after a match, record an implicit feedback signal: positive if the next user input is a continuation command (yes/y/continue/etc per kernel line 181); negative if it's an `igap absent-instruction` event within 2 turns OR a verbatim re-statement of a similar prompt. Auto-tune logic at lines 214-236 already exists; this PR wires the input. Threshold update gated by `L:dispatch-auto-tune ≡ true` (default false). Test: 20 fake matches with 8 negatives, assert threshold tightens by 0.05.
- **Depends on:** PR-002
- **Why:** Auto-tune code is wired but the feedback log is empty (zero entries since 2026-05-04). Either the feedback path is not real or it needs an automatic recorder. This PR provides the recorder.
- **Spec:** 03-prs/PR-014.md (not written yet)

## PR-015 — Audit trail for auto-edits
- **Status:** not-started
- **Complexity:** M
- **Phase:** C — Self-improvement plumbing
- **Scope:** New tool `tools/auto_audit.py`. Schema: `{ts, actor: "orchestrator"|"cron"|"hook", action, target, before_hash, after_hash, before_excerpt, after_excerpt, gate_passed, rule_invoked}`. Every auto-applied change writes an entry to `my-axon/log/auto-edits/YYYY-MM-DD.md` (append-only, daily file). New program `workspace/programs/auto-actions.md` reads recent entries and renders a recap table. Surfaces on next boot if any entries since last view.
- **Depends on:** PR-001 + PR-002
- **Why:** Trust is the gate to enabling autonomy. Without an audit trail, the user can't verify what happened overnight; without verification, autonomy stays off. This PR is the trust foundation for PR-017.
- **Spec:** 03-prs/PR-015.md (not written yet)

## PR-016 — Rollback for auto-edits
- **Status:** not-started
- **Complexity:** M
- **Phase:** C — Self-improvement plumbing
- **Scope:** Extend `tools/memory.py`'s 3-version L: rollback pattern to: (a) compiled programs — before any auto-recompile, save current `*.cmp.md` to `workspace/programs/compiled/.rollback/<name>-<ts>.cmp.md`, keep last 3; (b) W: keys — before each auto-edit, snapshot to `workspace/memory/working/.rollback/`. Single `tools/undo.py` becomes the lookup point: `undo --target compiled/<name>` or `undo --target W:<key>` reverts to N-back. Test: auto-edit a compiled program, undo, assert restored.
- **Depends on:** PR-009
- **Why:** Auto-edits without rollback is high-risk. With it, the worst case is "ran an unwanted optimization, ran undo, fine". Three-version cap keeps storage bounded.
- **Spec:** 03-prs/PR-016.md (not written yet)

## PR-017 — Auto-improvement orchestrator
- **Status:** not-started
- **Complexity:** L
- **Phase:** D — Orchestrator + UX
- **Scope:** New program `workspace/programs/auto-improve.md` registered as a daily cron entry. On run, gates: (1) `L:auto-improve ≡ true` (default false, owner-set via dev-mode), (2) drift state ≤ medium per PR-012, (3) for any kernel-affecting fix: `L:dev-mode ≡ true`. Three narrow auto-actions: (a) **auto-compile** programs reaching ≥5 uses in last 7d that aren't compiled (calls `compile rank --auto`), (b) **auto-tune** dispatch threshold per PR-014's feedback log when neg-rate > 30% over 20 dispatches, (c) **auto-archive** episodic memory entries > 30d to compressed annual archive. Each action: PR-015 audit entry, PR-016 rollback snapshot. Anything else (kernel patches, LANG extensions, new tools, program-content edits beyond compile) stays human-confirmed. Surfaces a one-line "Auto-improvements applied: N" at next boot.
- **Depends on:** PR-010, PR-011, PR-012, PR-014, PR-015, PR-016
- **Why:** This is the headline. Phase 1 study confirmed all infrastructure exists; what was missing is the orchestrator that closes loops. Three narrow categories chosen because all are regenerable / single-value / non-destructive — building trust before expanding scope.
- **Spec:** 03-prs/PR-017.md (not written yet)

## PR-018 — Menu + output-layer UX
- **Status:** not-started
- **Complexity:** M
- **Phase:** D — Orchestrator + UX
- **Scope:** `workspace/programs/menu.md` — surface meta tools (axon-audit, compile-optimizer, axon-docs-gen, auto-actions) under a new "Self-improvement" panel. Show: drift state badge, last cron run + next due, count of pending auto-actions, last 3 audit entries with link. `axon/OUTPUT-LAYER.md` — drift score line item (post-PR-012 it has decision impact, so visibility is now warranted). New compiled `menu.cmp.md` after lands.
- **Depends on:** PR-012, PR-015, PR-017
- **Why:** Without visibility, autonomy feels like loss of control. The panel makes "what is AXON doing for me" answerable in one glance.
- **Spec:** 03-prs/PR-018.md (not written yet)

## PR-019 — Inter-tool shared library
- **Status:** not-started
- **Complexity:** L
- **Phase:** D — Orchestrator + UX
- **Scope:** New `tools/_axon_lib.py` exporting frequently-shelled-to functions (compile.format, dispatch.match, usage.record, events.emit, kv-store.get/set). Refactor 5+ tools that today subprocess to other tools (compile-suggest → compile-write, axon-audit → multiple, dispatch-stats → dispatch). Keep CLI surface identical. Pytest: every existing tool test still passes.
- **Depends on:** PR-002, PR-007
- **Why:** Subprocess hops cost ~50ms each and serialize JSON, slowing the orchestrator. With 5 hops in axon-audit alone, that's quarter-second wall on every audit. Library imports remove the cost without changing the user surface.
- **Spec:** 03-prs/PR-019.md (not written yet)

## PR-020 — Programs in REGISTRY
- **Status:** not-started
- **Complexity:** M
- **Phase:** D — Orchestrator + UX
- **Scope:** Either: (a) extend `tools/REGISTRY.json` with a `programs:` section, or (b) split into `tools/REGISTRY.json` + `workspace/programs/REGISTRY.json` (per-area). Each program: `{name, file, status: ACTIVE|DEPRECATED|EXPERIMENTAL, area, exec_count, last_modified}`. Auto-populated from program headers via a `regenerate-registry` cron job (weekly). Replaces the per-program filesystem scan that runs today on every menu / status / dispatch.
- **Depends on:** none (but most useful after PR-019 so the orchestrator can read it via shared lib)
- **Why:** Today every program's metadata lives in its own header; menu/status/find-program scan all 66 files on every render. With 130+ programs (predicted by year-end with library-dev growth), this becomes a real cost. Registry is the standard fix.
- **Spec:** 03-prs/PR-020.md (not written yet)

---

## Sequencing dependency graph

```
PR-001  ──┬──>  PR-009  ──>  PR-016  ──┐
          │                              │
PR-002  ──┼──>  PR-003                   │
          │                              │
          ├──>  PR-006                   │
          │                              ├──>  PR-017  ──>  PR-018
PR-005 ───┼──>  PR-004                   │       (the
          │                              │       headline)
          ├──>  PR-007                   │
          │                              │
          └──>  PR-014  ─────────────────┤
                                         │
PR-001 ───>  PR-010  ──>  PR-011  ───────┤
                                         │
PR-002  ──>  PR-012  ────────────────────┤
                                         │
PR-001 ──┐                               │
PR-002 ──┴──>  PR-015  ──────────────────┘

PR-008 (independent)
PR-013 (independent)
PR-019 (depends PR-002 + PR-007; orthogonal to autonomy)
PR-020 (mostly independent; valuable after PR-019)
```

Critical path: **PR-001 → PR-002 → PR-015 → PR-017** (the autonomy spine). Everything else is parallelizable around it.
