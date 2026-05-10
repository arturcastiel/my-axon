# Study — axon-3.0
Updated: 2026-05-08  ·  Phase 1 — Comprehensive system audit  ·  No plan yet

## Goal

Map AXON v3.0.0 in full: every kernel rule, program, tool, and signal store. Identify inconsistencies, failure points, redundancy, usability gaps, and (the load-bearing question) **whether AXON can drive its own improvements automatically based on observed usage**.

This is a thorough, codebase-grounded study — not a plan. All findings cite paths and line numbers. Synthesized from four parallel explorers covering kernel, programs, tools, and observability/self-improvement signals.

---

## Layer 0 — System map

```
/mnt/c/projects/axon/  (AXON_ROOT, public, axon.git)
│
├── axon/                   ← OS core (write-gated, L:dev-mode required)
│   ├── KERNEL-SLIM.md      ← v1.1.0, 647 lines · 12 Core Rules + 11 gates
│   ├── BOOT.md, HOWTO.md, DEVELOPER.md (v1.1.1), COMMANDS.md, OUTPUT-LAYER.md
│   ├── core/    LANG.md (v2.3.0, 14 EXT) · TRANSLATE.md · OUTPUT.md · RUN-HEADER.md
│   ├── compiler/    COMPILER.md (v1.1.0) · GRAMMAR.md · templates/
│   ├── memory/   MEMORY.md (W:/L:/E: scopes)  · scheduler/SCHEDULER.md · processes/PROCESS.md
│   ├── log/LOG.md · programs/ (12 OS-level program specs) · tools/ (12 tool specs, no .py)
│   └── archive/KERNEL-LEGACY.md (v0.1.0, archived 2026-05-02)
│
├── tools/                  ← 50 Python tools (49 ACTIVE + 1 OPTIONAL after lint-paths add)
│   ├── _axon_paths.py      ← canonical path resolver (PR-001 this session)
│   ├── REGISTRY.json       ← single source of truth for tools
│   └── *.py                ← kernel | os | code-dev | documentation categories
│
├── workspace/              ← OS config (tracked, public)
│   ├── programs/           ← 66 .md programs + compiled/ (59 .cmp.md)
│   ├── preferences/        ← agent.md (inference-mode locked at 3) · output-layer.md · smart-dispatch.md
│   ├── scheduler/          ← cron.json (1 entry, never fired) · queue.json (2 done, 0 active)
│   ├── events/             ← event-log.json (2 events recorded)
│   └── log/, memory/        ← mostly empty (runtime, gitignored after .gitignore update)
│
├── my-axon/                ← user data (private, gitignored from axon.git, separate my-axon.git)
│   ├── MYAXON.md           ← path map (per-machine, allowed by feedback memory)
│   ├── dev-projects/       ← 3 projects: axon-os, opm-common, axon-3.0 (this study)
│   ├── memory/             ← longterm/ (13 keys), episodic/ (session-log), local/ (gitignored both ways)
│   ├── log/entries/        ← 8 daily log files
│   ├── log/igap/           ← 4 igap files (5 unresolved gaps from 2026-05-06)
│   └── chats/, plans/, libraries/, generated/
│
├── scripts/copilot/        ← Copilot persona slot files (this session)
├── scripts/setup-persona.sh   ← Claude Code Output Style installer
├── .github/copilot-instructions.md, .github/workflows/ci.yml
├── .vscode/settings.json   ← Copilot slot wiring
├── .pre-commit-config.yaml ← path-lint hook
├── .copilotignore, AGENTS.md, COPILOT.md, CONTRIBUTING.md
└── startup.md, SETUP.md, README.md, CHANGELOG.md, CONTEXT.md, WORKFLOW.md
```

**Counts**: 50 tools · 66 programs (59 compiled = 89%) · 59 axon/ kernel files · 14 LANG extensions · 11 enforcement gates · 12 Core Rules.

---

## Layer 1 — Kernel: rules, gates, language

### Core Rules (KERNEL-SLIM lines 63–76)

12 immutable rules, all enforced. Severity hierarchy: **higher number wins** (Rule 8). Three are `!CRIT`: identity, write-gate (Rule 9), cognition-language (Rule 11). Rule 12 (menu always-rendered) is enforced by output-layer; Rule 4 (always log) is partially enforced (some programs skip).

### Enforcement gates (KERNEL-SLIM §COMPLIANCE ENFORCEMENT, lines 79–308)

| Gate | Trigger | Bypass? | Wired in code? |
|---|---|---|---|
| Identity gate | "what are you / who made you" | NO | Mechanical via TOOL(verify) |
| Response gate | Every output | NO | R_REASONING_TRACE in verify.py |
| Cognition-language gate (G-02) | Before every reasoning step | NO | ASSERT(L:cognition-frame) |
| Write gate | WRITE/APPEND to axon/ | NO | enforce.py check-write |
| No-queue rule | After write gate refusal | NO | Strict, kernel line 166 |
| Active-program interrupt | Every user input | NO | K/I/A prompt, kernel line 168 |
| Arithmetic gate | Float/money/% | NO | Static, Core Rule 3 |
| Confidence gate | CONFIDENCE(n) < 0.7 | Soft | LOG(WARN) + QUERY |
| Inference gate | Before QUERY | Soft | L:inference-mode (3, locked) |
| IGAP tracker | Post-response | NO (!BG) | igap.py record (4 trigger types) |
| Context pressure gate | Before phase transition | Skippable | context.py status |
| Inference-mode lock | STORE(L:inference-mode, *) | Only via dev-mode | KERNEL-SLIM lines 270–275 |
| Halt mode | Gate failure | Configurable | strict (default) / soft |

All gates trace to a real code path. Gates run, mostly. The notable exception: **drift score is computed every turn but never read by any decision logic** (drift.py records, output-layer renders the badge, nothing else acts on it).

### Language layer (LANG.md v2.3.0)

- 26 base symbols, 50+ core ops, 14 ACTIVE extensions (EXT-001 through EXT-014).
- **EXTEND protocol** (LANG §SELF-IMPROVEMENT, lines 225–282): defines how new ops graduate from PROPOSED → ACTIVE after 2 successful uses. Manual; no auto-promotion.
- **Compiler grammar gap**: EVAL/RETRY/TEE are defined in LANG but have **no natural-language → op rules in compiler/GRAMMAR.md**. Compiler emits warnings on these ops; users must write the symbolic form directly. Concrete: "evaluate output against criteria" → no rule; "retry if fails" → no rule.
- **Compiler Phase 3 (OPTIMIZE)** is truncated at line 99 — incomplete spec.

---

## Layer 2 — Programs: 66 total, 59 compiled

### Inventory by area

| Area | Count | Compiled | Notes |
|---|---|---|---|
| chat | 1 | 1 | chat-input |
| build/run/test | 4 | 4 | mode-detect, mode-router, mode-suggest, run-tests |
| memory | 4 | 3 | memory-compact (compiled), resume, undo, show-memory (uncompiled) |
| code-dev | 11 | 10 | All but `code-dev-pr-review` compiled |
| **library-dev** | 9 | **0** | **Entire family uncompiled — biggest coverage gap** |
| axon meta/audit | 6 | 6 | axon-audit, axon-compare, axon-docs-gen, health-check, compile-optimizer, suggest-compile |
| tools/discovery | 5 | 5 | discover, gain, deps, find-program, list-tools |
| system/ops | 7 | 4 | menu, status, stats compiled. faq/glossary/versions are reference docs (intentionally uncompiled) |
| igap | 1 | 0 | igap-improve uncompiled |
| harness | 2 | 2 | harness-builder, register-tool |
| workspace ops | 4 | 0 | my-axon-init, migrate-workspace, save-workspace, workspace-backup — one-shots |
| reference | 2 | 0 | authoring-guide, translate (intentional) |
| other | 5 | 5 | handoff, session-summary, turn-log, output-layer, new-chat |

Header compliance is **100%** — every program has `# PROGRAM:`, `# desc:`, `## INSTRUCTIONS`, `## OUTPUT`. The `code-dev-pr-spec missing OUTPUT` flag from compile-optimizer test-all is a **false positive** in test-all's parser (the program does have OUTPUT semantics; the section heading uses non-standard formatting).

### Coupling

Two large fan-out hubs:
- `code-dev` → 8 children (init, study, plan, pr, pr-review, explain, log, audit)
- `library-dev` → 9 children (new, ingest, explain, intersect, search, report, cite, status)

Self-loops: `quickstart` recurses on step counter. `igap-improve` cascades through study → plan → pr.

No circular dependencies detected. DAG-shaped.

### Convention drift across programs (cosmetic, non-functional)

- **CHECKPOINT**: 23/66 use it after STORE; rest don't. Inconsistent for resume safety.
- **IDENTITY LOCK** preamble: ~12 programs declare it; rest don't.
- **STORE(W:active-phase, ...)**: 43 use phase tracking explicitly; 21 don't. Affects interrupt-resume granularity.
- **Help format**: dual conventions (`## HELP` + tag block vs `## PURPOSE` + `## INPUTS`). Parsed by both compiler and `explain`, so neither is broken.
- **Dispatch style**: code-dev uses `IF cmd ≡ "X"`; library-dev uses `IF subcommand ≡ "Y"`. Bikeshed-level naming drift.

### Discoverability gaps

- **Meta tools (axon-audit, compile-optimizer, axon-docs-gen) not on the menu** — only surfaced via direct typing.
- **`code-dev-pr-spec` vs `code-dev-pr` vs `code-dev-pr-review`** — three programs whose names don't sufficiently distinguish them (spec writes the spec, pr drives the PR cycle, pr-review reviews an existing PR).
- **`igap improve` shorthand** vs **`igap-improve` program file** — naming inconsistency.
- **`code-dev-shadow`** is an inspection mode but presented as a top-level command. Belongs as `code-dev status --shadow` or similar.

### Duplicate/overlap candidates

| Pair | Verdict |
|---|---|
| code-dev-pr vs code-dev-pr-spec | **Merge candidate** — both produce PR specs; spec is OPM-flavoured. Single program with `--style` flag suffices |
| code-dev-explain vs library-dev-explain | Distinct — code diffs vs academic articles. Keep separate |
| discover vs gain | Complementary — waste-finder vs longitudinal analytics |
| health-check vs axon-audit | Complementary — tool registry vs structural integrity |
| save-workspace vs workspace-backup | **Merge candidate** — both push my-axon to remote |
| my-axon-init vs migrate-workspace | One-shots, both rare. Could nest migrate inside init |

---

## Layer 3 — Tools: 50 total, REGISTRY-driven

### Adoption of `_axon_paths` (this session's PR-001)

- **38/50 tools import `_axon_paths`** (refactored).
- **12 holdouts**: `_axon_paths.py` (the resolver itself), `calculator.py`, `clock.py`, `diff_tool.py`, `document_parser.py`, `rtk.py`, `shadow.py`, `test_runner.py`, `tokenizer.py`, `translate.py`, `validator.py`, `web_search.py`. Most are stateless utilities (no workspace path needed). `shadow`, `document_parser`, `web_search`, `test_runner` *should* migrate for consistency.

### Probe coverage in `health.py:PROBES`

- **13 functional probes** (calculator, clock, tokenizer, diff, validator, kv-store, notify, semantic-search, document-parser, web-search, translate, verify, drift) — exercise actual behavior.
- **36 smoke-only probes** — only run `--help`. No regression detection beyond import errors.
- Highest-impact gaps: `compile-write`, `dispatch`, `usage`, `events`, `igap`, `compile-optimizer`, `axon-audit` — all critical to self-improvement loops, all smoke-only.

### Tool consolidation analysis

| Consolidation | Recommendation | Reason |
|---|---|---|
| **events + hooks** | **MERGE** | Events are the bus; hooks listen. Single tool with subcommands clearer. **1 tool saved**. |
| **compile-write + compile-suggest + compile-optimizer** | Consider | Three tools for three phases of one pipeline. Subcommand style: `compile {format,rank,optimize,verify}`. **2 tools saved**, but cohesive code. |
| **dispatch + dispatch-stats** | Keep separate | Different cadence (real-time match vs batch report) |
| **usage + prompt-log + igap** | Keep separate stores, merge CLI | Distinct signals (frequency, queries, gaps) but redundant CLI surface; one logger with `--type` flag |
| **memory + kv-store** | Keep separate | Different semantics (scoped+rollback vs flat cache) |
| **process + queue** | Keep separate | Process tracks one flow; queue manages many |
| **verify + enforce + validator + axon-audit** | Partial merge | verify+enforce → `check`. validator (JSON-schema) and axon-audit (structural) stay separate |

**Net consolidation potential**: 3 tools eliminated (events+hooks, two compile- tools), CLI surface tightened on logging trio.

### Stdlib replacements (small wins)

- `clock.py` → `datetime.now().isoformat()` if NTP isn't actually used. Currently keeps NTP path as fallback. Worth keeping.
- `process.py` (40 lines) — could fold into `queue` or `run`. Marginal.

### Missing tools

- **No general lint** beyond `lint_paths` (this session). Could grow to AST-pass argparse defaults, etc.
- **No `test-coverage`** report tool.
- **No `docs-validate`** for docgen output completeness.
- **No `perf-profile`** beyond `benchmark`.

### Inconsistencies in tool authoring

- **Argparse style**: subparsers vs positional `action=` arg, mixed across tools. Standardize on subparsers.
- **JSON output shape**: errors live under `"error"`, `"message"`, `"errors"` (validator), or non-JSON exit codes. No canonical envelope.
- **Stderr vs stdout for errors**: most tools dump JSON to stdout regardless; some log human text to stderr. Inconsistent.
- **Atomic writes**: `queue_tool.py` uses temp+rename (correct). `memory.py`, `events.py` write directly. Race-window.
- **Subprocess invocation between tools**: at least 5 tools shell out to other tools (compile-suggest → compile-write, etc.). No shared library; argument parsing duplicated.

---

## Layer 4 — Self-improvement: signals vs actions

### Signals AXON records about itself

| Store | Path | Volume now | Written by | Read by (auto) |
|---|---|---|---|---|
| Daily log entries | workspace/log/entries/, my-axon/log/entries/ | 1 + 8 files | log.py (every program) | None auto |
| IGAP gaps | workspace/log/igap/, my-axon/log/igap/ | 0 + 4 files (5 gaps) | igap.py record (4 triggers in kernel) | None auto |
| Session-log | my-axon/memory/episodic/session-log.md | 6 lines | session-save.py | boot reads on resume |
| Usage log | my-axon/memory/longterm/usage-log.jsonl | 18 entries | run.py auto | usage.py top (manual), compile-suggest (manual) |
| Dispatch index | my-axon/memory/longterm/dispatch-index.json | 59 entries | compile-write.py | dispatch.py match |
| Dispatch feedback | dispatch-feedback.jsonl | **0** (file absent) | dispatch.py feedback (never invoked) | dispatch.py auto-tune (lines 214–236, never fires) |
| Prompt log | memory/episodic/prompt-log.jsonl | **disabled** | prompt-log.py (toggle off) | pattern.py cluster (manual) |
| Drift trace | drift-trace.json | empty | drift.py record (manual init only) | drift.py check (manual) |
| Cron schedule | workspace/scheduler/cron.json | 1 entry: health daily @09 | cron.py add | **never fired** (run_count=0) |
| Event log | workspace/events/event-log.json | 2 events | events.py emit | hooks.py (manual) |

### Wired loops vs designed loops

| Loop | Design | Auto-fires? | Gate |
|---|---|---|---|
| **Usage → compile** | Record → suggest → compile | NO | Manual `compile-suggest suggest`; auto-flag exists but no cron |
| **IGAP → improve** | Record (auto) → report → improve cascades to study/plan/pr | NO | Every step requires QUERY(user); no cron entry |
| **Dispatch feedback → tune threshold** | Feedback → auto-tune at neg_rate>30% | Wired but dormant | Feedback log empty (never invoked) |
| **Pattern → compile** | Prompt-log → cluster → suggest | NO | Prompt-log disabled by default |
| **Drift → halt/replan** | Compute score → block divergent execution | NO (signal only) | drift score computed but unread |
| **Memory compaction** | Archive old E: > 30d | NO | memory-compact program never invoked |

**No closed loop fires automatically.** All require explicit user invocation. The cron infrastructure exists (cron.py, scheduler/cron.json) but **the cron daemon has never executed a job** — `run_count=0` for the only scheduled entry.

### Concrete inactivity example

May 6, 2026: 5 inference gaps recorded in `my-axon/log/igap/2026-05-06.md`, type `fallback-exec`, missing-instruction context "shadow update". Each was **manually annotated as resolved** in the log file; **igap-improve never fired**, no auto-action taken. The full study→plan→execute cascade is wired (igap-improve.md lines 39–184) but requires manual invocation. Compare to spec: it should be `cron weekly: igap improve --days 7`.

### Risk surface for autonomy

If self-improvement loops were enabled to fire automatically, current guardrails:

- ✓ Write gate (axon/ writes require dev-mode)
- ✓ Verify rules (kernel-rule predicates at compile + response time)
- ✓ Halt-mode strict default (HALT on gate failure)
- ✗ No granular audit trail (suggest-compile.py's `E:auto-compile-log` referenced but never archived)
- ✗ No drift-based halt (drift score computed but not read by any decision)
- ✗ No rollback on auto-edits (memory.py keeps 3 versions for L: keys; nothing for compiled programs or W: keys)
- ✗ No daily summary of auto-actions taken

### Compaction / handoff state

- **`session-save.py`** writes `last-session-summary.md` (summary works) and `last-session-snapshot.md` (**0 lines — broken**).
- W: key restoration on resume is therefore not happening. Boot's "interrupted session" prompt offers continue/restart/skip, but the actual W: state being resumed is empty.
- Memory compaction program exists, never triggered.

---

## Layer 5 — Inconsistencies (cross-cutting)

| # | Where | Detail | Severity |
|---|---|---|---|
| 1 | HOWTO.md lines 22, 35 | References `KERNEL.md` as full boot file — file was archived to `archive/KERNEL-LEGACY.md` on 2026-05-02 | **HIGH** (stale doc) |
| 2 | KERNEL-SLIM line 2 vs DEVELOPER.md line 112 | Header says v1.1.0; changelog claims v1.1.1 was the latest bump | MEDIUM |
| 3 | KERNEL-SLIM lines 241–268 (igap gate) | Kernel-explorer flagged "igap not in REGISTRY"; **incorrect — verified ACTIVE**. Same for context. Kernel-explorer's claim was stale. | (corrected) |
| 4 | LANG v2.3.0 vs compiler/GRAMMAR.md | EVAL/RETRY/TEE defined in LANG; no natural-language grammar rules. Compiler emits warnings, falls back to literal EXEC | MEDIUM |
| 5 | compiler/COMPILER.md line 99 | Phase 3 OPTIMIZE truncated; spec ends mid-section | MEDIUM |
| 6 | last-session-snapshot.md | 0 lines despite session-save.py path | MEDIUM (silent broken handoff) |
| 7 | drift score | Computed every turn, read by zero decision logic | LOW (signal-only) |
| 8 | dispatch-feedback.jsonl | File never created; auto-tune dormant | LOW |
| 9 | prompt-log enabled flag | Default `false`; pattern→compile loop hard-gated off | LOW |
| 10 | tools/cron.py:165 | TypeError on `data["jobs"]` — pre-existing, not from this session's refactor. Cron check broken from CLI | MEDIUM |
| 11 | code-dev-pr-spec | Compile-optimizer flags missing OUTPUT section; false positive due to non-standard heading style | LOW |
| 12 | library-dev/* | 0/9 compiled vs code-dev's 10/11. Coverage gap for the entire family | MEDIUM |

---

## Layer 6 — Failure points

| Scenario | Consequence | Mitigation |
|---|---|---|
| KERNEL-SLIM.md missing/corrupt | Identity gate cannot ASSERT; reasoning falls back to prose (Core Rule 11 violation); !CRIT halt cascade | None at boot — file is mandatory |
| tools/REGISTRY.json missing | Tool mandatory check fails; verify.py cannot run rule predicates; response gate fails | Error; no fallback |
| my-axon/MYAXON.md missing | W:myaxon-* keys uninitialized; programs hit ∅ on RETRIEVE; QUERY user repeatedly | Boot offers `[F]resh / [C]lone / [S]kip` (handled) |
| Cron daemon not running | All scheduled work dormant (currently the case — health daily 09:00 has 0 runs) | Latent, no alarm |
| Context compaction loses L:cognition-frame mid-program | Cognition-language gate fails on next reasoning step | G-02 mid-loop re-assertion every 5 turns + boot restore |
| Compiler hits LANG op without grammar rule | Warning emitted; falls back to literal EXEC; loses optimization | Tolerable; compiler does not crash |
| `tools/cron.py check` raises TypeError | Cron status check unusable from CLI | Pre-existing bug, surfaces on every reboot |
| session-save snapshot empty | Handoff to next session restores no W: state | Resume offers menu instead of program continuation |
| LANG.md version drift between LANG and TRANSLATE | New op renders as raw symbol to user (Rule 7 violation) | None — manual sync |

---

## Layer 7 — Improvement opportunities (themed, no plan)

### A. Reliability / consistency

1. **Fix `tools/cron.py:165`** — current TypeError makes cron status invisible. Pre-existing bug; small fix.
2. **Fix `last-session-snapshot.md` write path** in session-save.py — currently produces 0-line file.
3. **Stale HOWTO.md ref** to KERNEL.md → KERNEL-SLIM.md.
4. **Version stamp sync** between KERNEL-SLIM header and DEVELOPER.md changelog.
5. **Standardize tool argparse to subparsers** across the suite.
6. **Canonical JSON error envelope** for tool output.
7. **Atomic-write all kv stores** (memory, events). Add temp+rename pattern.

### B. Coverage / completeness

1. **Compile the library-dev family** — 0/9 → at least suggest, intersect, explain (highest-frequency).
2. **Compile uncompiled meta**: igap-improve, code-dev-pr-review, code-dev-shadow.
3. **Functional probes for kernel tools**: compile-write, dispatch, usage, events, igap, axon-audit (currently smoke-only).
4. **Fill compiler grammar gap**: EVAL/RETRY/TEE natural-language rules in compiler/GRAMMAR.md.
5. **Complete COMPILER.md Phase 3 (OPTIMIZE) spec** — currently truncated.

### C. Tool consolidation (saves 3+ tools)

1. **events + hooks → unified event system** (single tool with `--listen`/`--emit` subcommands).
2. **compile-write + compile-suggest + compile-optimizer → `compile {format,rank,optimize,verify}`** (one tool, clearer pipeline).
3. **save-workspace + workspace-backup → one program**.
4. **code-dev-pr + code-dev-pr-spec → `code-dev pr --style {default,opm,detailed}`**.

### D. Usability

1. **Surface meta tools on menu**: axon-audit, compile-optimizer, axon-docs-gen.
2. **Rename `code-dev-pr-spec`** to disambiguate from `code-dev-pr` and `code-dev-pr-review`.
3. **Promote `axon` shorthand** in menu for one-key access to common operations.
4. **Show drift score** in menu when state is non-stable.

### E. Self-improvement autonomy (the headline question)

The infrastructure is **observe-only today**. To shift toward auto-improvement:

1. **Activate the cron daemon**: it's wired but dormant. Without it, every other auto-loop stays manual.
2. **Schedule weekly self-improvement runs** (compile-suggest, igap report, dispatch-stats, memory-compact).
3. **Wire drift score into decision logic**: high drift → halt; medium drift → confidence penalty in output layer.
4. **Enable prompt-log opt-in** in agent.md preferences once trust is established.
5. **Auto-tune dispatch threshold** (already wired; needs feedback recorded — start by integrating feedback into dispatch flow itself).
6. **Add audit trail for any auto-edit** (compiled program rewrite, threshold adjustment, kernel patch). Surface to user once per session.
7. **Rollback on auto-edits**: extend `memory.py`'s 3-version rollback to compiled programs and W: keys.
8. **Daily "what AXON did automatically" report** — visible accountability.

### F. Architectural

1. **Shared library between tools** instead of subprocess invocation (reduces duplicated arg parsing across 5+ tools).
2. **REGISTRY.json as the single source for program metadata too**, not just tools (currently each program's metadata lives in its own header).
3. **Output-layer drift visibility**: badge already exists; nothing acts on it.

---

## Layer 8 — The headline question

> **Can AXON build and improve its own workflow automatically based on usage?**

**Today: NO. Architecturally: YES, with the work in Layer 7 §E.**

What's true now:
- ✓ AXON observes itself richly (usage, igap, dispatch, prompt-log, drift, events, session-log).
- ✓ All improvement loops are designed and wired in code.
- ✓ Guardrails exist (write-gate, dev-mode, halt-mode strict, identity gate).
- ✗ No loop fires automatically — every step ends in `QUERY(user)` or requires manual invocation.
- ✗ Cron infrastructure exists but the daemon has never executed a job (`run_count=0`).
- ✗ Drift score is read by zero decision logic.
- ✗ Session handoff snapshot is silently empty.

What's missing for autonomy:
- An **active scheduler** that fires registered cron entries (or a boot-time check that surfaces overdue self-improvement work).
- **A weekly cycle**: `compile-suggest → igap report → dispatch-stats → memory-compact → session-summary`. Today, all manual.
- **Drift gating** as a real decision input.
- **Audit + rollback** for any auto-applied change (so trust can be earned incrementally).

This is achievable as Phase 2 work (no plan yet, per directive). The system was clearly architected toward this — every signal store, every tool, every gate is in the right shape. What's missing is the orchestrator that closes each loop without human-in-the-loop confirmation, and the trust gates that make doing so safe.

---

## Confidence

**AXON understanding: 8/10** — Comprehensive map across kernel, programs, tools, signals. Gaps: didn't trace the full compile-optimizer execution path against a real test corpus; didn't verify all 11 gates fire under contrived violation; didn't audit shadow file usage in code-dev programs deeply.

**User satisfaction: TBD** — awaiting confirmation. If insufficient, refine on: (a) deeper compile-optimizer trace, (b) rule-by-rule gate verification, (c) shadow-file mechanics, (d) library-dev coverage gap analysis, (e) per-tool LOC + dependency matrix.

---

## Sources

- Four parallel Explore agents covering: kernel rules + gates + language, program library + compiled coverage + duplicates, tool suite + adoption + consolidation, observability + self-improvement signals.
- Direct code reads: tools/REGISTRY.json (igap/context status verified), tools/cron.py (TypeError reproduced), workspace/scheduler/cron.json (run_count=0).
- This session's prior work: PR-001 through PR-008 (path-resolution refactor), PR-009 through PR-016 (Copilot integration + CI gates).

## Architecture Snapshot

(Populated in Phase 2 after detailed module boundary analysis — not done yet, per directive.)

## Open Questions

1. Is the cron daemon meant to run on every boot, or only when explicitly started? KERNEL-SLIM doesn't say. If boot-only, every fresh session pays catch-up cost.
2. Should `library-dev` family be compiled given how rarely it's used today, or kept uncompiled until usage grows?
3. Is the `code-dev-pr-spec` (OPM-style) variant still needed, or can it be folded into `code-dev-pr` with a flag?
4. Should drift score gate execution (refuse to run a diverged program) or just modify confidence?
5. What's the appetite for autonomous auto-edits to compiled programs versus keeping all kernel-affecting changes human-confirmed?
