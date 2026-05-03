# STD-005: Test Code Work Product Standard

**Purpose:** Define what a test code deliverable must look like, so that quorum participants producing tests and reviewers evaluating them have a shared understanding of the deliverable.

---

## What Test Code Is

Test code is a separate library that verifies the application code against the test plan. It imports from the application, exercises its interfaces, and asserts expected behavior. Test code lives in its own directory, separate from the application code.

## Relationship to Other Work Products

Test code is the final work product in the sequence:

1. **Functional Requirements** (STD-001) — what the system must do
2. **HLD** (STD-002) — how the system is architected
3. **Test Plan** (STD-003) — what to test and what success looks like
4. **Application Code** (STD-004) — the implementation
5. **Test Code** (STD-005) — the executable verification

Test code requires both the test plan (what to verify) and the application code (what to import and call). It cannot be produced until both exist.

## What Test Code Is NOT

- It is not the test plan. The test plan defines scenarios at a behavioral level. The test code implements those scenarios as executable assertions.
- It is not part of the application code. Test code does not ship with the application. It lives in a separate directory with its own dependencies.

---

## Deliverable Structure

A test code deliverable is a standalone test library. It contains:

1. **Test modules** — Organized to mirror the test plan's component breakdown. Each module covers a component, flow, or phase from the test plan.

2. **Test fixtures** — Shared setup, teardown, and test data. Reusable across test modules via shared fixture files (e.g., `conftest.py` in Python, shared helper modules in Node.js).

3. **Integration test infrastructure** — Configuration, compose files, or scripts needed to stand up an isolated test environment (see §4).

4. **Helpers module** — Shared utilities for interacting with the system under test (MCP calls, Docker commands, database queries, HTTP helpers). All test infrastructure code lives here, not scattered across test files.

5. **Dependency specifications** — Test-specific dependencies (test framework, mocking libraries) pinned to exact versions.

---

## 1. Test Architecture: Two Layers

Every test suite must have two distinct layers:

**Mock tests (unit tests)** — Test logic with mocked dependencies. Cover error paths, edge cases, input validation, pure computation, and routing logic that cannot be triggered against live infrastructure. Run anywhere, no infrastructure needed. Use the test runner and mocking framework specified in the test plan (STD-003 §2.5).

**Live tests (integration tests)** — Test against a deployed instance of the system. Exercise the full stack: real databases, real LLM calls, real pipeline processing. Verify end-to-end behavior, data flow, and system interactions.

Both layers are required. Mock-only suites miss integration bugs. Live-only suites are slow and can't cover error paths. The two layers complement each other.

### Organization

The test directory structure mirrors the test plan's component breakdown. The specific conventions (file naming, fixture patterns, marker registration) follow the language and test runner specified in the test plan (STD-003 §2.5).

General structure:

```
tests/
  mock/                        # Unit tests with mocked dependencies
  live/                        # Integration tests against deployed stack
    helpers.js or helpers.py   # All infrastructure helpers
  browser/                     # Browser automation tests (if applicable)
```

Mock tests use the standard conventions of the project's test runner. Live tests are organized into phases that reflect dependency order (infrastructure first, then tools, then pipelines, then quality).

---

## 2. No Skips, No Excuses

**2.1** Tests must not use skip mechanisms (`pytest.skip()`, `test.skip()`, `describe.skip()`, or equivalent) to avoid testing real functionality. If a test cannot pass, the code under test has a bug — fix the bug, don't skip the test.

**2.2** The only acceptable use of skip is when a genuine external precondition is absent (e.g., a required GPU is not available on the current machine). The skip message must state what is missing and how to provide it.

**2.3** LLM non-determinism is not a valid reason to skip a test. Design the test to accommodate non-determinism: check for behavioral outcomes (did the tool get called?), not exact text matches. Use broader assertions. If the LLM consistently fails to do what it should, that is a bug in the prompt, temperature, or system design — fix it.

---

## 3. Test Every Failure

**3.1** Every test failure must be investigated. A failing test is a finding. Never dismiss failures as "pre-existing," "unrelated to current work," or "known flaky."

**3.2** If a test run surfaces a bug outside the scope of the current task, log it as an issue and investigate it. The test suite exists to find bugs — ignoring the ones it finds defeats the purpose.

**3.3** Server errors (500s, connection failures, timeouts) in test output are bugs, not noise. Investigate the root cause in the server logs.

---

## 4. Isolated Test Infrastructure

When the system under test is a deployed service (containerized application), integration tests must run against an isolated instance that does not interfere with production or other test runs.

**4.1 Isolation requirements:**
- Separate Docker Compose project name (e.g., `-p test-prefix`)
- Separate network namespace
- Separate data volumes or data directories
- Separate configuration directory (with test-appropriate settings)
- Separate port bindings (no conflicts with production)

**4.2 Data loading idempotency:**
- The test fixture must check if data is already loaded before loading it again. Reloading 10,000+ messages on every test run is unacceptable.
- Check a count or marker, skip loading if data is present, load only if the database is empty or below threshold.

**4.3 Stack lifecycle:**
- The session fixture should check if the test stack is already running before deploying.
- Tearing down the stack after tests is optional — leaving it running allows faster iteration.
- The fixture must handle both fresh-deploy and already-running scenarios gracefully.

**4.4 Baseline reset between test sections:**

Tests can poison each other's state — accumulated data, open modals, stale caches, orphaned processes. When this can happen, the test suite MUST include a shared baseline reset script that returns the system under test to a known clean state. Every test section calls this reset before running.

The reset script is a shared module (not copy-pasted into each test). It is application-specific — each project defines what "clean state" means for its UI/API/database. Examples:
- Browser tests: close tabs, dismiss overlays, invalidate render caches, reload state
- API tests: delete test-created records, reset configuration to defaults
- Database tests: truncate test tables, reset sequences

The principle: every test must succeed or fail on its own merit, never because a previous test left debris.

**4.5 Visual verification for browser tests:**

Browser tests that check DOM state (CSS classes, element existence, attributes) can pass while the UI looks broken. Every browser test suite MUST capture screenshots that can be reviewed for visual correctness.

**Screenshot capture:** Each test produces a screenshot saved to a shared location (e.g., `/storage/screenshots/{project}/`). Filenames encode suite and test name: `{suite}--{test_name}.png`.

**Automated visual review:** After a test run, a lightweight model (e.g., Haiku) reviews each screenshot against two inputs:

1. **Test-specific expectations** — the reviewer reads the test code to understand what the test claims to verify, then checks the screenshot matches that intent.

2. **Generic visual checklist** — regardless of what the test is testing, check for:
   - Broken layout (overlapping, clipped, or missing elements)
   - Wrong font sizes (too big, too small, inconsistent with settings)
   - Error messages visible in terminal, status bar, or console
   - Modals or overlays blocking content when they shouldn't be
   - Test data pollution (junk project names, orphaned sessions, debug text)
   - Empty areas where content should exist
   - Unreadable text (contrast, color, truncation hiding meaning)
   - Scrollbars where there shouldn't be, or missing where needed
   - Inconsistent theme (dark elements in light theme or vice versa)

**Rating:** Each screenshot gets OK, WARNING, or PROBLEM. Problems are filed as issues. The visual review is part of the test pipeline — tests don't pass the gate if visual problems exist.

---

## 5. Testing Non-Deterministic Systems (LLM, Agents)

Systems that use LLMs produce non-deterministic outputs. Tests must be designed for this reality.

**5.1 Behavioral assertions over textual assertions:**
- Assert on outcomes: did the tool get invoked? Did the database row get created? Did the file appear on disk?
- Do not assert on exact LLM response text. Do not use brittle keyword lists.
- Count-before/count-after patterns work well for verifying side effects through layers of non-determinism.

**5.2 LLM-as-judge for quality evaluation:**
- When the quality of generated text matters (summaries, search relevance, agent coherence), use an LLM judge to evaluate the output.
- The judge receives the output and a rubric, and returns a structured rating (e.g., GOOD / ACCEPTABLE / POOR) with reasoning.
- The judge's reasoning is logged so failures can be diagnosed. The test asserts on the rating.
- The judge model should be more capable than the model being evaluated.

**5.3 Multi-turn agent testing:**
- Test agent conversations as sequences of turns, not individual calls.
- Verify that context carries across turns (the agent remembers what was said).
- Verify that tool calls produce real side effects (files created, database rows written, schedules persisted).

---

## 6. Pipeline and Worker Testing

Systems with background processing pipelines (embedding, extraction, assembly, etc.) require verification at each stage, not just the final output.

**6.1 Stage-by-stage verification:**
- After data ingestion, verify each pipeline stage completed: embeddings generated, extraction ran, assembly produced summaries.
- Query the database directly to verify intermediate state (embedding vectors exist, extraction marks set, summary rows created).

**6.2 Pipeline completion polling:**
- Use a polling pattern with timeout to wait for background workers to complete.
- Poll on observable indicators: queue depths reaching zero, row counts stabilizing, metrics counters stopping.
- Define a stall timeout — if no progress for N seconds, the pipeline is stuck.

**6.3 Performance measurement:**
- Scrape application metrics (Prometheus `/metrics`, custom endpoints, or timing logs) after the test run.
- Parse available metrics to estimate percentiles (p50, p90, p99) where applicable.
- Generate a performance report with latencies per tool, queue depths, job completion rates.
- Performance reports are artifacts of the test run, not assertions (unless specific SLOs are defined).

---

## 7. Gray-Box Testing

The standard "test through public interfaces only" guidance does not apply to integration testing of complex systems. Applications with databases, queues, and background workers require verification beyond their API responses to catch entire categories of bugs.

**7.1 What gray-box testing means here:**
- Query the database directly to verify state changes (rows created, columns updated, indexes present).
- Inspect Docker container logs for errors, warnings, and expected log entries.
- Query application metrics to verify counters incremented, histograms populated (where applicable).
- Check filesystem state inside containers (files created, permissions correct).

**7.2 When to use gray-box vs black-box:**
- Black-box (MCP tool call + response check): sufficient for simple request/response tools.
- Gray-box (tool call + database/log/metrics verification): required for tools that trigger background processing, modify persistent state, or have side effects beyond the response.

---

## 8. Runtime Issue Logging

Tests will discover issues that are not hard failures but are worth recording — degraded behavior, unexpected warnings, quality concerns, performance anomalies.

**8.1** Runtime findings are filed as GitHub Issues on the project repository using the `gh` CLI or GitHub API. This provides traceability, assignability, and persistence across sessions.

**8.2** Each issue includes: test name, severity label (bug/warning/performance), category label, description with expected vs actual behavior, and the test output or evidence.

**8.3** Issues at severity `bug` should also fail the test. Issues at `warning` severity are informational findings that do not fail the test but are filed for review.

**8.4** Tests should check for existing open issues before creating duplicates. Use `gh issue list --label` to check if a known issue already exists for the same failure.

---

## 9. Hot-Reload Testing

For systems that support runtime code deployment or configuration hot-reload, the test suite must verify that it works.

**9.1** Test that runtime code deployment successfully installs a package and the new code takes effect without container restart. For agentic services and pluggable agents, see STD-006.

**9.2** Test that configuration hot-reload (mtime-based file re-read) picks up changes to config files, prompt files, and credential files without restart.

---

## 10. Fixture Patterns

**10.1 Session-scoped fixtures** for expensive operations: deploying the test stack, loading bulk data, waiting for pipeline completion. These run once per test session, not per test. Implementation depends on the test runner (e.g., `before()` in `node:test`, session-scoped fixtures in pytest).

**10.2 Per-test setup** for isolated test state: creating a test conversation, generating a unique ID, resetting state. Implementation depends on the test runner (e.g., `beforeEach()` in `node:test`, function-scoped fixtures in pytest).

**10.3 HTTP client** scoped to the session, pointing at the test stack's base URL with appropriate timeout settings.

**10.4 Fixture ordering:** Infrastructure verification (health checks, readiness) must complete before any data loading or functional testing begins.

---

## 10a. Structural Coverage

**10a.1** The test suite must produce a structural coverage report using the coverage tool specified in the test plan (STD-003 §2.5).

**10a.2** Coverage is measured by running the full mock test suite through the coverage tool. Live and browser tests are excluded from structural coverage measurement (they verify integration, not code paths).

**10a.3** The coverage report must meet the minimum threshold defined in the test plan (STD-003 §2.6). The engineering gate does not pass if coverage is below threshold.

**10a.4** Coverage reports are test artifacts — committed or stored alongside test results for traceability.

---

## 11. Failure Investigation Discipline

When tests fail, the test code author (or the agent running the tests) must follow `process/PROC-001-debugging-guide.md`. This is a requirement, not guidance.

Key principles (see PROC-001 for the full step-by-step process):

- **Root cause before fix.** Never change code based on a guess. Trace the failure to its root cause with evidence — actual errors, actual data, actual code paths.
- **3-CLI independent analysis.** For every non-trivial bug, consult all three CLIs (Claude, Gemini, Codex) independently for root cause analysis and code review. Do not proceed with fewer than 3 independent analyses.
- **Never adjust tests to accommodate bugs.** The fix goes in the application code. No skips, no broadened assertions, no lowered thresholds, no "flaky" labels. If the test is wrong, fix it to be more correct — not less strict.
- **Fixes must be precise.** Address the root cause without side effects. No broad pattern matching, no silent reclassification, no workarounds that trade one bug for another.
- **Verify every fix.** Hot-reload or redeploy, run the failing test, then the full suite. An unverified fix is not a fix.

---

## Review Criteria

When reviewing a test code deliverable, check:

- **Test plan coverage:** Does every scenario in the test plan have corresponding test code?
- **Two-layer architecture:** Are there both mock tests and live integration tests?
- **No skips:** Are there any skip calls (`pytest.skip`, `test.skip`, `describe.skip`, or equivalent)? If so, each must have a justified reason that is not "the code doesn't work."
- **Failure investigation:** Were all test failures from the last run investigated? Are there unexplained failures being ignored?
- **Isolation:** Do integration tests run against an isolated instance? Is there any risk of interfering with production data?
- **Non-determinism handling:** For LLM/agent tests, do assertions check behavioral outcomes rather than exact text? Is LLM-as-judge used where quality matters?
- **Pipeline verification:** For systems with background processing, are intermediate stages verified, not just final output?
- **Gray-box coverage:** Are database state, logs, and metrics verified where appropriate?
- **Issue logging:** Does the suite log runtime findings for review?
- **Investigation discipline:** Were all failures from the last run investigated to root cause? Were second opinions sought for non-trivial bugs? Were fixes verified?
- **Independence:** Do tests run independently of each other? No test depends on another test's side effects.
- **Clarity:** Can a developer read a test and understand what behavior it verifies?
- **Practicality:** Do the tests actually run? Are infrastructure dependencies documented?
- **Imports real code:** Every test imports from the application. No local reimplementation of business logic inside test files. If a test would still pass with the application code deleted, it is not a test.
- **Error path coverage:** Every error handling path (catch blocks, error callbacks, rejection handlers) has an explicit test that triggers the error condition and verifies the observable outcome (log entry, error response, state change). Bare catch blocks that were remediated must each have a corresponding test proving the error is now handled.
- **Browser console capture:** Playwright/browser tests must capture and assert on console errors. A test that passes while the browser console shows uncaught exceptions is hiding bugs.
- **Regression validation:** For a sample of critical modules (at least 3), temporarily break the application code and confirm the corresponding tests fail. A test suite that passes against broken code is worthless.
- **Fixture-driven inputs:** Test data comes from shared fixture files or factories, not hardcoded inline in test bodies. This enables reuse across test modules and makes test data maintenance tractable.
- **Structural coverage:** The test suite produces a coverage report meeting the threshold defined in the test plan (STD-003 §2.6). Coverage is measured on mock tests only.
- **Suite structure:** The test file layout matches the structure defined in the test plan (STD-003 §16).
