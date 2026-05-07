# PLAN: smart-dispatch
Goal:    Automatically detect frequent prompts, routines, and workflows; suggest them for compilation; match incoming requests to compiled versions and route to the cheaper compiled execution path
Created: 2026-05-04
Updated: 2026-05-04
Status:  complete

## TASKS

### Phase 1 — Observation (data collection)
- [x] P1-1: Extend `usage.py` to record full prompt hashes and free-text labels alongside program run counts
- [x] P1-2: Build `usage suggest` output: top N uncompiled high-frequency patterns flagged for compilation
- [x] P1-3: Add session-level prompt logging (opt-in) — capture raw user inputs to `workspace/memory/episodic/prompt-log.md`

### Phase 2 — Pattern recognition
- [x] P2-1: Build `tools/pattern.py` — cluster prompt-log entries by semantic similarity (TF-IDF + KMeans) and surface recurring themes
- [x] P2-2: Define threshold policy: suggest compilation when a workflow appears ≥ N times (default N=3, configurable via prefs)
- [x] P2-3: Output ranked suggestion list: name / frequency / estimated token savings / compile command

### Phase 3 — Compilation integration
- [x] P3-1: Build `tools/compile-suggest.py` — wraps pattern.py + usage suggest + compiler prompts into one "suggest and offer to compile" flow
- [x] P3-2: Registered compile-suggest in REGISTRY.json; dispatch index updated on compile via register_in_dispatch_index()
- [x] P3-3: Add `auto-compile` preference flag in smart-dispatch.md — if true and --auto passed, compile silently when threshold crossed

### Phase 4 — Smart dispatch (routing to compiled)
- [x] P4-1: Build `tools/dispatch.py` — given a user prompt, compute TF-IDF cosine similarity against dispatch index; return best match + confidence
- [x] P4-2: Dispatch policy: confidence ≥ dispatch-confidence pref (default 0.65) → dispatch; else → fallback: agent
- [x] P4-3: Wire DISPATCH PRE-FLIGHT section into COMMANDS.md — pre-flight check before agent reasoning
- [x] P4-4: dispatch.py logs confidence + match to feedback log; usage.py records dispatched runs

### Phase 5 — Feedback loop
- [x] P5-1: `dispatch feedback --id ID --result yes|no` — records outcome in dispatch-feedback.jsonl
- [x] P5-2: Auto-tunes dispatch-confidence: ≥30% negative rate in last 20 dispatches → +0.05 to threshold
- [x] P5-3: Build `tools/dispatch_stats.py` — `summary --window 7d` shows N dispatches, tokens saved, accuracy, compile candidates

## NOTES
Key tools involved: usage.py · pattern.py · compile-suggest.py · dispatch.py · dispatch-stats.py
All 5 phases complete. 4 new tools registered in REGISTRY.json.
COMMANDS.md wired with DISPATCH PRE-FLIGHT block.
Dispatch index lives at: workspace/memory/longterm/dispatch-index.json
