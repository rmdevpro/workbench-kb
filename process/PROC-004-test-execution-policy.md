# PROC-004 — Test Execution Policy

This is the canonical policy for how Workbench tests are selected, executed, and recorded. PROC-003 (runbook execution guide), the master runbook, and the test scope matrix all defer to this document for the rules below.

## Principle 1 — Scope is decided by the matrix, not by humans at runtime

The test scope matrix is the **only** mechanism that determines which gates and which phases run for a given change type. The matrix is authored at design time, not chosen during execution.

- Orchestrators do not pick what runs. Executors do not pick what runs. The matrix decides; the orchestrator briefs the executor with the resulting set.
- "Phase 0 fresh-container setup is not needed for this run" is not a SKIP. It's the matrix saying that phase is out of scope for the change type.
- A phase is either in scope or it isn't. Once it's in scope, every test in it runs.

## Principle 2 — SKIP is not a valid result

There is no per-test orchestrator skip authority. There is no per-test executor skip authority. There is no "exercised-elsewhere" footnote that converts an unrun test into a PASS.

- A test that doesn't run because the matrix excluded it doesn't appear in the result set at all. It is not recorded as SKIP.
- A test that runs and fails is FAIL, with a GitHub issue. It is not recorded as SKIP.
- A test that runs and passes is PASS, with explicit assertions specific to that test. It is not recorded as SKIP.

There is no fourth state.

## Principle 3 — Every tool call is a test of that tool

This applies to executors and to orchestrators equally.

When a tool you use fails or returns an unexpected shape:
1. **File a GitHub issue immediately.** Before the next tool call.
2. **Inform the orchestrator** (or the user, if you are the orchestrator) what failed.
3. **Continue the run** via a legitimate alternate signal that does not mask the failure.

What "legitimate alternate signal" means: a signal that exists for the purpose at hand, through a code path that is not the one being tested. Polling the results file when `session_read_screen` is broken is legitimate (file mtime is independent of the MCP screen handler). Falling back to a deprecated tool that bypasses the failure is not legitimate — it converts a tool-failure-with-issue into a hidden tool failure.

## Principle 4 — Stop only on pollution

A failed test does not justify stopping the run. The default is **file → continue.**

Stopping is justified only when the failure **pollutes downstream tests** — corrupted state that would produce false signals in tests that follow. Examples:

- Phase 1 (Smoke) fails: app isn't reachable, every later test would FAIL spuriously → STOP, file, report.
- A session-creation test corrupts the DB state used by session-listing tests later → STOP, file, repair, restart.
- A single MCP tool returns the wrong shape but doesn't corrupt anything → file, continue. Other tests can still run truthfully.

When in doubt, continue. Pollution must be demonstrable, not hypothetical.

## Principle 5 — Expediency belongs to the gate, not to the test set

Gates encode the time/depth tradeoff:

- **Gate A** (mock/unit) — fast. Run on every change.
- **Gate B** (live integration) — slower. Run when the change touches behavior, not just code shape.
- **Gate C** (browser acceptance) — slowest. Run when the change affects what the user sees.

The matrix uses gate selection to manage time. "We don't have time" is answered by selecting fewer gates per the matrix, never by skipping tests within a selected gate.

## Principle 6 — One test ID, one explicit result, one set of assertions

No aggregate PASS that bundles multiple tests under a single result line. No "I tested A, B, and C in one go" entries. Each test has its own:

- ID
- Specific assertions (what shape, what content, what side effects)
- Explicit result line (PASS or FAIL)
- Issue reference if FAIL

If a tool is not directly tested by an explicit assertion in the gate where it lives, it is **untested**, regardless of what other tests indirectly touch.

## Principle 7 — Same rules for executor and orchestrator

The orchestrator is not exempt from any of the above. If the orchestrator's own tool calls fail, the same response applies: file → continue → no silent workaround. The orchestrator does not have authority that the executor lacks.

## What to do with this policy

- **Authoring tests:** every test entry has an explicit assertion list. No aggregates. No "exercised-elsewhere" claims.
- **Running tests:** brief the executor with exactly the in-scope phases per the matrix. The executor records PASS or FAIL on every entry. SKIP is not on the menu.
- **Encountering tool failures:** file before the next tool call. Continue via a non-masking alternate signal. Do not stop unless pollution is demonstrable.
- **Reviewing results:** any aggregate result, "exercised-elsewhere" footnote, or unexplained SKIP is a process failure, not a passing run.
