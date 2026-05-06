# Workbench UI Test Runbook — Execution Guide

This guide is for the **orchestrator** who runs the master UI test runbook (`tests/workbench-test-runbook.md`) against a Workbench deployment. It covers spawning an executor, briefing it, monitoring it, recovering from common failure modes, and triaging the results.

The runbook itself is a test catalog — what to test. This guide is the procedure — how to drive it to completion.

## Roles

- **Orchestrator** — you. Decides target, spawns the executor, monitors, intervenes, triages results, files issues, applies fixes.
- **Executor** — a separate Claude session created and managed via the Workbench `session_*` MCP tools. Reads the runbook, runs each test using its tools, writes results to a separate file. Has no decision-making authority over what to skip.

Two distinct conversations. Don't conflate them. Never use a subagent (`Agent` tool) — the executor must be a real session with full Playwright + Bash + filesystem access.

## Phase 1 — Decide the target

Bind these placeholders before doing anything else:

- `${WORKBENCH_URL}` — base URL of the deployment under test
- `${WORKBENCH_CONTAINER}` — Docker container name on the host
- `${WORKBENCH_HOST}` — `user@host` reachable via ssh for `docker exec` etc.
- `${GATE_USER}` / `${GATE_PASS}` — gate credentials if the deployment is gated; empty otherwise

Confirm scope per **PROC-004 (Test Execution Policy)** and the test scope matrix. Phases excluded by the matrix for this change type **do not appear in the briefing at all** — they are not "skipped," they are out of scope. There is no per-test SKIP authority for the orchestrator or the executor.

Common matrix-driven exclusions:

- Phase 0 (Fresh Container + Auth) — out of scope when the matrix says we are reusing an existing dev container with persistent auth, not skipped at runtime.

## Phase 2 — Spawn the executor

Write the briefing file first (see below), then create the executor session using `session_new` with a short initial prompt pointing at it. The executor session gets the same tools any Workbench Claude session has: Playwright MCP, Bash, filesystem access.

```
session_new {
  project: "<a-project-name>",
  cli: "claude",
  prompt: "Read tests/executor-briefing-<date>.txt and follow it exactly."
}
  → {session_id: "<executor-session-id>"}
```

Capture the `session_id`.

### Briefing — what to put in the briefing file

Write the briefing to a file in the workspace (e.g., `tests/executor-briefing-<date>.txt`), then start the session with a short initial prompt pointing the executor at it:

```
session_new {
  project: "<a-project-name>",
  cli: "claude",
  prompt: "Read tests/executor-briefing-<date>.txt and follow it exactly."
}
  → {session_id: "<executor-session-id>"}
```

This avoids `session_send_text` size limits and keeps the full briefing as a committed artifact alongside the results file.

Concrete, with no decisions left to the executor. Cover at minimum:

1. **Target binding** — every placeholder from Phase 1, with concrete values
2. **Tools available** — Playwright MCP for UI, curl for API, ssh + docker exec for infra. Make it explicit: there is no test you cannot run. Per PROC-004, SKIP is not a valid result — every in-scope test runs and is recorded as PASS or FAIL with a filed issue on FAIL.
3. **Output file** — exact path where to write results (`tests/runbook-results-<date>.md`); format per test
4. **Edit-append, not Write-whole-file** — every test result added via the `Edit` tool against a stable anchor like `## Issues Filed`. Do NOT rewrite the entire file each time. This is the single most important durability rule
5. **Per-test commit/push cadence** — every N tests, `git add` + commit + push so partial progress survives executor failure
6. **Out-of-scope phases** — list them by name with a reference to the scope matrix entry that excluded them. These phases are not in the executor's working set; the executor never records SKIP. Per PROC-004, SKIP is not a valid result.
7. **Issue Filing Protocol** — point at the runbook's section. Each FAIL produces a GitHub issue
8. **Pacing** — Phase 1 (Smoke) must fully pass before any other phase runs; if it fails, stop and report

### Verify the briefing landed

```
session_read_screen {session_id: "<executor-session-id>"}
```

You should see the executor reading the runbook and the results file (if a partial exists), then beginning tests. If it's stuck at the prompt without output, the briefing didn't deliver — resend via `session_send_text` + `session_send_key "Enter"`.

## Phase 3 — Inline polling

### The orchestrator is an agent, not a lookup table

The orchestrator is an agent applying judgment in real time. The job is **continuous monitoring of the executor and removal of any blockage that prevents forward progress**, regardless of whether the blockage matches a pattern documented below. The intervention table further down is a list of common patterns we've seen — it is **examples, not a complete contract**.

### No progress is never acceptable

**It is never acceptable for the executor to be making no progress.** If two consecutive checks show zero forward motion (no new test results in the file, no token growth on screen, no new tool calls), that is a blockage. Diagnose and clear it. Do not "wait and see." Do not assume long thinking time is normal. The default state is forward motion; anything else is the orchestrator's signal to act.

### Foreground polling only — never rely on background processes

Polling must be **synchronous in the orchestrator's own session**. Use foreground waits (`session_wait`, inline shell sleeps that you immediately follow with a check) so progress verification is on the same execution path you're driving from. **Never rely on background processes** (`Monitor`, `run_in_background`) for progress checks — events from those drift, arrive late, and let the executor sit in a no-progress state without the orchestrator noticing. If you started a wait, the next thing you do is check progress; the wait is not a license to walk away.

Novel failure modes (executor stops at an empty prompt without crashing, executor produces a result but never commits, executor hits an unfamiliar dialog, executor enters a state nothing in this doc anticipates) are the orchestrator's to diagnose and clear. Reach for the right tool given the situation, file an issue if a tool fails, send the right input to the executor to unstick it, and keep it moving until it reaches Phase 15 or hits a true terminal state.

If the doc doesn't cover the situation, the answer is not "wait for the doc to be updated." The answer is to use judgment, then update the doc once the run is done so the next orchestrator has the pattern.

### Polling cadence

The polling loop is **synchronous in the orchestrator's own session**. Sleep, then check, then decide. Do not use background tools (`Monitor`, `run_in_background`) — events from those don't arrive reliably enough to react in real time, and the system blocks long bare sleeps anyway.

The standard pattern (5-minute cycle):

```
# 1. Wait
session_wait {session_id: "<executor-session-id>", seconds: 270}

# 2. Check executor screen
session_read_screen {session_id: "<executor-session-id>"}

# 3. Check result file progress (via Bash)
wc -l tests/runbook-results-<date>.md
grep -cE '^\*\*Result:\*\* (PASS|FAIL)' tests/runbook-results-<date>.md
```

`270` (4.5 min) keeps the orchestrator within the prompt cache window; bump up only if context permits.

### What to watch for at every tick

- **Progress signals**
  - Token count increasing on the spinner line
  - Result file line count growing
  - New `**Result:** PASS|FAIL` blocks appearing
  - Recent pane output mentions test IDs you'd expect

- **Freeze signals** (act if confirmed across two ticks)
  - Token count flat
  - Same pane content captured
  - File mtime unchanged
  - Spinner still showing but no tool calls landing

- **Context-window-saturation signals** (act immediately)
  - Pane shows `Context limit reached` or `Auto-compacting conversation`
  - Pane shows `Tip: Use /clear to start fresh when switching topics and free up context`
  - Token count climbing past the model's effective working ceiling (well before the hard limit)
  - Tool calls failing with `Error writing file` or similar truncation symptoms

The context-saturation case is the silent killer. The executor will eventually hit the limit, auto-compact (losing detail), and start producing degraded output before fully crashing. Catch it early.

### Common intervention patterns (examples — not exhaustive)

The patterns below are common cases we've seen. They are starting points for diagnosis, not a complete decision tree. Apply judgment for any symptom not listed.

| Symptom | Intervention |
|---|---|
| Executor working but not flushing results to disk | `session_send_text` a clear instruction: write everything you have via `Edit` NOW, then continue per the rules. Follow with `session_send_key "Enter"` |
| Queued message not being processed (executor stuck mid-turn) | `session_send_key "Escape"` to interrupt the current turn, then `session_send_key "Enter"` to submit the queued message |
| Executor frozen — token count flat across two ticks, screen unchanged | Escape + Enter as above. If still frozen on next tick, kill and spawn a fresh executor (next row) |
| Executor finished its turn and is idle at empty prompt (looks frozen on file-mtime alone, but screen shows the default empty-state suggestion) — i.e., it stopped instead of continuing into the next phase | `session_send_text` a continuation prompt: which phase/test ID to resume from, the same Edit-append + commit rules, and an explicit instruction not to stop until Phase 15 or context-saturation. Follow with `session_send_key "Enter"`. Distinguishing this from a true freeze requires reading the screen; without screen visibility, file mtime alone is insufficient. |
| Context-window saturation imminent | `session_send_text "/clear"` + `session_send_key "Enter"`, wait for the prompt to clear, then re-inject the briefing — including: prior results file path, where to resume (last completed test ID), what tests remain, and the same Edit-append rules |
| Executor hung past recovery, or fresh start makes more sense than untangling | `session_kill {session_id}`. Spawn a new executor via `session_new` with the partial results file as input and instructions to carry forward all prior PASSes, redo FAILs, run remaining tests |

The `/clear` + re-injection path matters most. When it works, the executor keeps the same session id (so any orchestrator-side references stay valid) but with a fresh context window. The re-brief must be self-sufficient — assume the executor has zero memory of the prior turn:

- Re-state the target binding
- Point at the partial results file
- Tell it explicitly which test ID to resume from
- Repeat the Edit-append + per-N-tests commit rules
- Repeat the out-of-scope-phase list (matrix-derived; not a SKIP list)

## Phase 4 — End of run

The executor declares done. Verify before accepting:

```bash
# Coverage check
runbook_test_count=$(grep -cE '^### (REG|NF|SMOKE|CORE|FEAT|EDGE|CLI|E2E|USER|SESS|EDIT|TASK|MCP|KEEP|QDRANT|PROMPT|CONN)-' tests/workbench-test-runbook.md)
results_count=$(grep -cE '^\*\*Result:\*\* (PASS|FAIL)' tests/runbook-results-<date>.md)
# Counts should align with the in-scope phase set; results may exceed if carry-forwards from a partial included extra HOTFIX-style entries.

# SKIP audit — per PROC-004, SKIP is not a valid result. Any SKIP in the results is a process failure.
! grep -q '^\*\*Result:\*\* SKIP' tests/runbook-results-<date>.md || echo "PROCESS FAILURE: SKIP results found"
```

For every FAIL, confirm a GitHub issue exists. If the executor missed any, file them now per the runbook's Issue Filing Protocol.

Commit and push the final results file if the executor didn't.

### File issues in series, not parallel

`gh issue create` calls in parallel can race — issue numbers in the response don't always match the test you intended. File them sequentially, capture each returned URL, then build the Issues Filed table from the verified numbers.

## Phase 5 — Triage FAILs

For each FAIL, classify into one of three buckets. The classification determines what closes the issue.

| Classification | Indicator | Action |
|---|---|---|
| Real product bug | Code does the wrong thing under documented expectations | Fix the code via PROC-01 → close issue with verification |
| Test bug | Code is correct; the test is testing for old/wrong behavior | Update the runbook; close the issue with "not a bug — test updated" |
| Removed feature | Test targets something deliberately deleted from the product | Delete the test entries from the runbook (keep the corresponding `REG-VOICE-01`-style "removed" check if one exists); close the issue with "not a bug — feature removed" |
| Feature gap | Runbook expects behavior that was never implemented | Re-frame the issue as `enhancement` (not `bug`); leave open for the next implementation round |
| Test infra gap | Test cannot run on the current target without a separate environment (e.g., needs a wipeable container for fresh-install tests) | Either build the missing infra or fold the test into a phase that already provides it (e.g., Phase 0.A handles fresh-install) |

Single FAIL can map to multiple of these — handle each.

## Phase 6 — PROC-01 per real bug

For each "real product bug" FAIL:

1. **RCA** — describe the symptom in one self-contained file (`/tmp/symptom-<id>.txt`). Send to three independent CLIs (Claude via `Agent`, Gemini, Codex) and collect their root-cause diagnoses. Look for ≥2-CLI consensus.

2. **Fix** — implement; keep the diff small and targeted

3. **UI test** — if the fix touches anything visible, exercise it via Playwright (or Hymie for OAuth/Ink-input flows). Backend-only verification of a UI-touching fix is insufficient

4. **Runbook entry** — confirm the existing REG-* entry now passes, or add one if none existed. Every fix needs a permanent regression test in the runbook, not a one-shot "verify this fix" entry

5. **3-CLI code review** — diff sent to all three CLIs again. Any concern flagged by ≥2 CLIs gets addressed before merge. Single-CLI concerns are noted but not auto-actioned (unless the concern is obviously a real bug)

6. **Re-verify** after addressing review feedback

7. **Close the issue** with a verification comment that includes: the commit SHA(s), the verification evidence, and the 3-CLI review summary

## Lessons that should always make it into the briefing

These are recurring failure modes from past runs. State them explicitly in every executor briefing:

- **Edit-append, not Write-whole-file.** Holding test results in memory until the end risks losing everything to context exhaustion. Each test gets one `Edit` against a stable anchor immediately
- **No SKIP at any level.** Per PROC-004, SKIP is not a valid result. Tools are always available; "I don't have X" is wrong, you do. Run every in-scope test; record PASS or FAIL with an issue on FAIL.
- **Commit + push every N tests** so partial progress survives executor failure
- **Read with offset+limit** — never load the entire runbook or the entire results file in one Read call
- **Never load huge tool outputs into context.** If a Bash command would print 1000+ lines, redirect to `/tmp/x` and `head` or `wc` it
- **Use the matrix-derived in-scope phase list verbatim.** Don't run out-of-scope phases; don't skip in-scope tests.

## Lessons for the orchestrator

- **Inline-poll only.** Background notifications drift; sync polling forces engagement. The orchestrator's own session must stay alive and attentive throughout
- **Watch for context saturation early.** By the time the executor hits the auto-compact line, you're already in recovery mode. Catch the climbing token count and `/clear` proactively if the run is long
- **File issues sequentially.** Parallel `gh issue create` calls race
- **Don't conflate the executor session with the test target.** The executor is a Workbench session you talk to via `session_*` tools; the test target may be a different deployment reached over HTTP/ssh. Keep those two interaction paths mentally separate — `session_read_screen` reads the executor, not the target
- **The runbook is the regression suite, not a one-shot checklist.** Every fix lands a permanent REG-* test, not a HOTFIX-* entry that becomes dead weight
