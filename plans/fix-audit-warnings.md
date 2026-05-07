# PLAN: fix-audit-warnings
Goal:    Resolve all 7 warnings surfaced by axon-audit — build 3 missing programs, fix 2 wrong EXEC paths, fix 1 auditor false positive, and decide on rtk
Created: 2026-05-04
Updated: 2026-05-04
Status:  complete

## BACKGROUND
Running `axon-audit` on 2026-05-04 returned:
  Verdict: WARNINGS  (36 OK · 8 WARN · 0 FAIL)

The 8 warnings break down into 4 categories:

  A) Missing programs (need to be built)
  B) Wrong EXEC paths (file exists, caller uses wrong name)
  C) Auditor false positive (doc example inside → "" string)
  D) Optional tool stub absent

---

## TASKS

### Category A — Build missing programs

- [x] A1: Build `workspace/programs/resume.md`
- [x] A2: Build `workspace/programs/register-tool.md`
- [x] A3: Build `workspace/programs/chat-input.md`
- [x] B1: Fix `axon/programs/new-chat.md` EXEC path → `EXEC(_chat-checkpoint)`
- [x] B2: Fix `axon/programs/switch-chat.md` EXEC path → `EXEC(_chat-checkpoint)`
- [x] B3: Fix `workspace/programs/mode-router.md` → `EXEC(plan-new)`
- [x] C1: Updated `tools/axon_audit.py` to strip `→ "..."` lines before EXEC_RE scan
- [x] D1: Built `tools/rtk.py` stub — returns not_installed JSON when rtk CLI absent

---

## PRIORITY ORDER
B3 (5 min, fixes live routing bug) → C1 (10 min, clean audit output) →
B1+B2 (10 min, fix axon/ EXEC paths) → A3 (chat-input, most used) →
A1 (resume, useful UX) → A2 (register-tool, dev workflow) → D1 (cosmetic)

## NOTES
- B1 and B2 require L:dev-mode ≡ true (writes inside axon/)
- A1/A2/A3 go in workspace/programs/ (no dev-mode needed)
- After all tasks: re-run axon-audit → expect 0 WARN, 0 FAIL
- rtk stub (D1) would raise ACTIVE count from 41 to 42 but health still shows SKIPPED for rtk
  since the script won't actually connect to anything useful
