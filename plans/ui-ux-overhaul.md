# PLAN: ui-ux-overhaul
Goal:    Design and implement a comprehensive UI/UX improvement across all AXON interaction surfaces — menus, output, mode system, onboarding, and runtime feedback
Created: 2026-05-04
Updated: 2026-05-04
Status:  active

## PHASES

### PHASE 1 — AUDIT & DESIGN  (this document)
- [x] Audit all current UI surfaces (menu, status, help, list-*, quickstart, output-layer)
- [x] Identify friction points and gaps
- [x] Write detailed design spec per surface
- [ ] Review design with owner

### PHASE 2 — CORE SHELL
- [ ] P2-1  Redesign menu: contextual dashboard, not just a mode picker
- [ ] P2-2  Redesign status: live OS dashboard (inference mode, memory pressure, ops trace)
- [ ] P2-3  Redesign output-layer: surfaced on every response (mode, ops, confidence, turn)
- [ ] P2-4  Add breadcrumb bar: shows active mode → active program → active step

### PHASE 3 — PROGRAM DISCOVERY
- [ ] P3-1  Redesign list-programs: grouped by category with compiled badge + usage rank
- [ ] P3-2  Redesign list-tools: show tool status inline + latency + last-used
- [ ] P3-3  Redesign help [program]: uniform 3-section format (WHAT / HOW / EXAMPLE)
- [ ] P3-4  Add find-program UX: fuzzy search with scored candidates

### PHASE 4 — ONBOARDING FLOW
- [ ] P4-1  Redesign quickstart: linear wizard with state tracking (not just output)
- [ ] P4-2  Add first-run checklist: health-check → compile → dispatch → first program
- [ ] P4-3  Add interactive tutorial mode: guided AXON session with inline prompts

### PHASE 5 — MODE SHELLS
- [ ] P5-1  Redesign each mode (chat/build/run/memory/system/plan/programs/dev)
         Each mode gets: entry banner, active context, inline help, exit hint
- [ ] P5-2  Add mode-status: per-mode status line shown inside mode
- [ ] P5-3  Add mode-suggest: after 3 turns in a mode, suggest related programs

### PHASE 6 — ERROR & FAILURE UX
- [ ] P6-1  Standardize FAIL() output: problem / cause / fix hint / suggested next command
- [ ] P6-2  Add recovery suggestions on QUERY(user) gates
- [ ] P6-3  Add "did you mean?" on unknown program names

### PHASE 7 — COMPILE & RECOMPILE
- [ ] P7-1  Redesign compile-optimizer output: progress bar, savings summary table
- [ ] P7-2  Add axon-diff: show what changed between source and compiled
- [ ] P7-3  Show stale compiled badge in list-programs when source is newer

---

## DETAILED DESIGN SPEC

---

### SURFACE 1: MENU  (current: basic mode list + tip)

**Problems:**
- No visible inference mode or OS state at a glance
- No compiled coverage or dispatch readiness shown
- Health only shown if L:health-score cached — not live
- Tip is random noise; could be contextual guidance
- Mode descriptions are one-liners, not interactive

**Proposed redesign:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  AXON  ·  {ws}                            {date}
  {tools} tools  ·  inf:{inf-mode}/10({inf-label})  ·  {outmode}
  {IF dev → "⚙ DEV MODE ACTIVE" | ""}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  OS STATE
  ────────────────────────────────────────────────
  Health      {bar}/100  {label}
  Compiled    {compiled}/{total} ({coverage}%)  ·  Dispatch: {dispatch} entries
  Inference   {inf-mode}/10 — {inf-label}
  Memory      {wkeys} working keys  ·  {lkeys} longterm keys
  {IF queue > 0 → "  ⚠ Queue  {queue} pending"}
  {IF cron > 0  → "  ⏰ Cron   {cron} scheduled"}

  ACTIVE WORK
  ────────────────────────────────────────────────
  {IF chat    → "  ◈ Chat    {folder}/{name} — {goal[:45]}"}
  {IF plan    → "  ◈ Plans   {count} active: {names[:3]}"}
  {IF resumed → "  ↺ Resumed {last-line}"}

  MODES
  ────────────────────────────────────────────────
  {▶ if active} [1] CHAT      — conversations + iteration
  {▶ if active} [2] BUILD     — write + compile programs
  {▶ if active} [3] RUN       — execute programs
  {▶ if active} [4] MEMORY    — browse + manage knowledge
  {▶ if active} [5] SYSTEM    — tools, health, session
  {▶ if active} [6] PLAN      — goals + milestones
  {▶ if active} [7] PROGRAMS  — discover + inspect
  {▶ if dev}   [D] DEV        — kernel editing

  QUICK COMMANDS
  ────────────────────────────────────────────────
  status · stats · help [prog] · quickstart
  {contextual suggestion based on last action}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### SURFACE 2: STATUS  (current: raw W: dump)

**Problems:**
- Dumps raw W: key/value map — no grouping, no priority
- No inference mode shown
- No ops trace (what just ran)
- Game state and investigation mixed with OS state

**Proposed redesign:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  AXON STATUS  ·  {date} {time}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  OS CORE
  ────────────────────────────────────────────────
  Inference mode  {inf}/10 — {label}  [{locked? "LOCKED" | ""}]
  Output mode     {outmode}
  Dev mode        {dev}
  Health          {score}/100 — {label}
  Turn            {turn}

  ACTIVE CONTEXT
  ────────────────────────────────────────────────
  Mode            {curmode | "none"}
  Program         {active-program | "none"}
  Chat            {active-chat | "none"}
  Confidence      {response-confidence | "—"}%

  MEMORY
  ────────────────────────────────────────────────
  Working keys    {COUNT(W:)} — {list of key names}
  Longterm keys   {COUNT(L:)} — {list}
  Episodic        {recent count} events

  LAST OPS TRACE
  ────────────────────────────────────────────────
  {W:last-ops | "(no ops recorded this turn)"}

  QUEUE / CRON
  ────────────────────────────────────────────────
  {queue items or "Queue: empty"}
  {cron jobs or "Cron: none scheduled"}

  RECENT EVENTS
  ────────────────────────────────────────────────
  {last 5 session-log events}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### SURFACE 3: OUTPUT LAYER  (current: confidence bar + drift + program + turn)

**Problems:**
- Inference mode never shown
- No ops trace on the response footer
- Dev mode not indicated
- Format is inconsistent (sometimes blank)

**Proposed standard footer (compact):**

```
▸ AXON  [{program}]  T:{turn}  inf:{inf}/10  ·  {drift-icon}  ·  {conf-bar} {conf}%
  ops: {W:last-ops | "—"}
──────────────────────────────────────────────────────────────
```

**Full format:**
```
┌─ AXON ──────────────────────────────────────────────────────┐
│  Program    : {program}          Turn : {turn}              │
│  Inference  : {inf}/10 — {label} {IF locked → "[LOCKED]"}  │
│  Confidence : {bar} {conf}%                                 │
│  Drift      : {drift-icon} (score: {drift.score})           │
│  Ops trace  : {last-ops | "—"}                              │
│  Dev mode   : {dev}                                         │
└─────────────────────────────────────────────────────────────┘
```

---

### SURFACE 4: LIST-PROGRAMS  (current: flat sorted list)

**Problems:**
- No compiled badge — can't see at a glance which are fast
- No usage stats inline
- No category grouping — 59 programs in one flat list
- No stale-compiled indicator

**Proposed redesign:**

```
▶ list-programs  ·  {filter | "all"}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  SYSTEM  (axon/programs/)                       12 programs
  ──────────────────────────────────────────────────────────
  ● interactive    [compiled]  7×   Continuous AXON session loop
  ● new-chat       [compiled]  2×   Create a new chat session
  ● list-programs  [compiled]  —    Catalog every program
  …

  WORKSPACE  (workspace/programs/)               47 programs
  ──────────────────────────────────────────────────────────
  ANALYSIS & AUDIT
  ● axon-audit     [compiled]  3×   Structural + usefulness audit
  ● axon-compare   [compiled]  1×   Compare AXON vs similar frameworks
  ● health-check   [compiled]  7×   Tool health probe
  ● stats          [compiled]  —    Workspace health dashboard

  COMPILATION & DISPATCH
  ● compile-optimizer [compiled] —  Token-saving compilation engine
  ● suggest-compile   [compiled] —  Surface compile candidates
  …

  LEGEND:  ● compiled  ○ source-only  ⚠ stale  ×N = usage count
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### SURFACE 5: HELP [program]  (current: varies by program)

**Problems:**
- Every program formats help differently
- Some have full tables; some just dump the source
- No standard WHAT / HOW / EXAMPLE layout

**Proposed standard format for all help:**

```
▶ help — {program-name}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  WHAT IT DOES
  {desc — 1-2 sentences, plain English}

  HOW TO RUN
  ────────────────────────────────────────────────
  {program-name}                     — default run
  {program-name} --input key=value   — with input
  {program-name} --section part      — if sectioned

  INPUTS  (optional unless marked *)
  ────────────────────────────────────────────────
  {key}*   {type}   {description}
  …

  OUTPUTS
  ────────────────────────────────────────────────
  {what gets written, where, what format}

  EXAMPLE
  ────────────────────────────────────────────────
  {a realistic 1-3 line example of running it}

  RELATED
  ────────────────────────────────────────────────
  {2-4 related programs with one-liner each}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### SURFACE 6: MODE ENTRY BANNERS  (current: bare)

**Problems:**
- Entering a mode gives no context: no commands, no current state, no exit hint
- No per-mode status line
- No suggestions

**Proposed mode entry banner (example: BUILD mode):**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ▶ BUILD MODE  ·  {ws}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Write, compile, and manage AXON programs.
  Compiled: {compiled}/{total} ({coverage}%)

  COMMANDS
  ────────────────────────────────────────────────
  new program [name]      — scaffold a new program
  compile [name]          — compile a program
  compile all             — compile all uncompiled
  suggest-compile         — find compilation candidates
  list-programs           — see all programs
  help [name]             — docs for any program
  simulate [name]         — dry run before executing
  diff [name]             — compare source vs compiled

  Type naturally or use a command above.
  '0' or 'menu' to return to the main menu.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### SURFACE 7: FAILURE & ERROR UX  (current: FAIL() raw message)

**Problems:**
- FAIL() messages vary; some are terse, some have no recovery hint
- No "did you mean?" on typos
- No structured cause / fix / next-step format

**Proposed FAIL() standard:**

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  ✗  AXON FAIL  ·  {program}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Problem  :  {reason — plain English, 1 sentence}
  Cause    :  {technical detail}
  Fix      :  {actionable 1-liner}

  Next steps:
    → {suggested command 1}
    → {suggested command 2}
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

### SURFACE 8: QUICKSTART  (current: 5-step linear output)

**Problems:**
- No state tracking — if user leaves mid-tour, it restarts
- Steps are output-only — user can't skip or jump
- No first-run checklist linking to concrete programs

**Proposed redesign:**
- Store step progress in W:quickstart-step
- Allow: "next", "skip", "jump to 3", "restart"
- End with a live checklist:
  ```
  YOUR AXON CHECKLIST
  [✓] health-check passed
  [✓] first program run
  [ ] first program compiled
  [ ] dispatch index populated
  [ ] first chat created
  ```

---

## IMPLEMENTATION ORDER  (recommended)

  Priority 1 (this week):
    P2-3  Output layer — inference mode + ops trace on every response
    P2-2  Status redesign — live OS dashboard
    P6-1  FAIL() standardization

  Priority 2 (next):
    P2-1  Menu redesign — OS state dashboard at top
    P3-1  list-programs — categories + compiled badge
    P5-1  Mode entry banners

  Priority 3:
    P3-3  Help standardization
    P4-1  Quickstart wizard with state
    P7-3  Stale compiled badges

  Priority 4:
    P4-3  Interactive tutorial mode
    P5-3  Mode-suggest (after 3 turns)
    P6-3  "Did you mean?"
