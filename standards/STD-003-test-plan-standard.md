# STD-003: Test Plan Standard

**Purpose:** Define what a test plan must contain, so that quorum participants producing test plans and reviewers evaluating them have a shared understanding of the deliverable.

---

## What a Test Plan Is

A test plan defines what to test, how to test it, and what success looks like. It translates the requirements and architecture into verifiable assertions about the system's behavior.

The test plan is produced before the code, not after. It is part of the anchor context for the code quorum — the models producing code know upfront what they will be tested against. The sequence is: Functional Requirements → HLD → Test Plan → Code → Execute.

## What a Test Plan Is NOT

- It is not a test framework selection guide. The test plan specifies tooling (§2.5) but does not evaluate or compare frameworks.
- It is not a code review. The test plan describes tests to write, not defects in the implementation.
- It does not re-debate requirements or architectural decisions.
- It is not written in response to code. It is written from the functional requirements and HLD before code exists.
- It is not a CI/CD automation definition. The test plan defines what gates must pass before deployment (§18), not how those gates are triggered. Do not prescribe specific automation tooling (GitHub Actions, Jenkins, etc.) or assume automated pipelines. Default to manual execution unless requirements state otherwise.

---

## Required Sections

### 1. Engineering Requirements Gate

The first step in any test plan. Before functional testing begins, verify compliance with the applicable engineering requirements: code formatting, linting, and unit test execution. These are prerequisites, not optional checks. These are REQ-001 (base) and, where applicable, REQ-002 (agentic services) and REQ-003 (pluggable agents). For standalone applications, these are REQ-001 plus any project-specific standards. See STD-006 for agentic service and pluggable agent additions.

### 2. Test Strategy: Two Layers

The test plan must define a two-layer test strategy:

**Mock tests (unit tests)** — Which components are tested with mocked dependencies? Define the mock boundary: what is mocked (databases, LLMs, HTTP calls, queues) and what is not. Mock tests cover error paths, edge cases, input validation, and logic that cannot be triggered against live infrastructure.

**Live tests (integration tests)** — Which components are tested against deployed infrastructure? Define what "deployed" means for this system: which containers, which backing services, what configuration. Live tests cover end-to-end data flow, pipeline processing, tool behavior, and system interactions.

For each test scenario, the plan must state whether it is mock or live, and why. A scenario that verifies database state after a tool call is live. A scenario that verifies input validation on a Pydantic model is mock.

### 3. Test Infrastructure

How the test environment is provisioned and isolated from production:

**2.1 Isolation strategy** — How the test instance is separated from production. For containerized systems: a separate compose file (e.g., `docker-compose.test.yml`) or a separate env file (e.g., `test.env` used with `docker compose --env-file test.env`) that provides different port bindings, volume paths, container names, and environment variables. The test container uses the same Dockerfile and application code as production — only the compose-level configuration differs (per REQ-001 §8.2). Both test and production containers can run simultaneously on different ports.

**2.2 Configuration** — What inference models, embedding models, and external services the test stack uses. For systems with LLM dependencies, define the test model set (fastest available models to minimize cost and latency).

**2.3 Data loading** — What test data is required, where it lives, how it is loaded, and how loading is made idempotent (so re-running tests does not reload data that is already present).

**2.4 Stack lifecycle** — How the test stack is started, how the test suite detects that it is ready, and whether the stack is torn down after tests or left running for iteration.

**2.5 Test tooling** — The test plan must specify the tools used for each testing concern. During plan genesis, propose the best modern equivalent for each category. During plan review, validate the choices are current and appropriate. The final plan locks the tooling for the test code deliverable (STD-005).

Required categories:

| Category | Node.js | Python |
|---|---|---|
| Test runner | `node:test` | `pytest` |
| Structural coverage | `c8` | `coverage.py` / `pytest-cov` |
| Browser automation | `playwright` | `playwright` |
| HTTP testing | `fetch` (native) | `httpx` / `requests` |
| Mocking | `node:test` mock API | `unittest.mock` / `pytest-mock` |
| Linting | `eslint` | `ruff` |
| Formatting | `prettier` | `ruff format` |

This grid is a reference baseline. Plan authors should update to the best modern equivalent at time of writing. Reviewers validate the choices are current — if a tool has been superseded, flag it during review.

**2.6 Coverage gating** — The test plan must define a minimum structural coverage threshold (line and branch) that the test suite must meet before the engineering gate passes. Coverage is measured by running the full mock test suite through the structural coverage tool. The threshold is project-specific but must be explicitly stated — "no threshold" is not acceptable.

### 4. Coverage Targets

What must be tested at minimum. The engineering requirements mandate happy path and common error conditions for all programmatic logic. The test plan identifies which specific paths those are for this system.

**4.1 Capability audit** — A complete enumeration of every capability the system provides (endpoints, tools, pipeline stages, worker behaviors, configuration options). Each capability is categorized: REAL (tested against live infrastructure), MOCK (tested with mocked dependencies), or NONE (not tested). The target is zero NONE entries.

**4.2 Coverage by layer:**
- Every API endpoint, MCP tool, or external interface must be tested with realistic inputs and verified outputs.
- Every background pipeline or processing stage must be verified.
- Every tool that produces side effects must be tested for real side effects, not just acknowledgment responses.
- Every configuration option that affects behavior must be tested.

### 5. Test Cases by Component

For each major component or flow, the specific scenarios to test:

- What is the input
- What is the expected outcome
- What conditions trigger the test (happy path, error, edge case)
- Whether this is a mock or live test
- What gray-box verification is needed (database state, log entries, metrics)

### 6. Endpoint and Tool Testing

Every external interface (MCP tools, HTTP endpoints, API surfaces) must be tested with full parameter variation — not just a single minimal-parameter happy path. Test with missing parameters, invalid inputs, boundary values, and realistic payloads. A passing test with a trivial input does not prove the endpoint works.

### 7. Pipeline End-to-End Verification

If the system contains processing pipelines (multi-stage flows), the test plan must define:

**6.1 Stage verification** — How each stage is verified independently. What database state, queue depth, or metric counter indicates that a stage completed successfully.

**6.2 Completion detection** — How the test suite knows the pipeline is done. Define the polling strategy: what indicator is polled, how often, what timeout, and what constitutes a stall (no progress for N seconds).

**6.3 Performance measurement** — What metrics are captured from the pipeline run. Define which metrics are collected (Prometheus, application logs, timing measurements), what percentiles are estimated, and whether a performance report is generated as a test artifact.

### 8. Non-Deterministic System Testing

If the system uses LLMs, agents, or other non-deterministic components, the test plan must define how non-determinism is handled:

**7.1 Behavioral assertions** — Define what behavioral outcomes are checked instead of exact text. For each LLM-dependent test, specify the observable side effect that proves the system worked (tool was invoked, database row created, file written).

**7.2 Quality evaluation** — For scenarios where output quality matters (summarization, search relevance, agent coherence), define the evaluation method. LLM-as-judge is the standard approach: specify the judge model, the evaluation rubric, the rating scale, and the minimum acceptable rating.

**7.3 Agent tool testing** — For tools that produce side effects, define how each is tested for real outcomes. "The agent said it wrote the file" is not verification — checking that the file exists on disk is. Define the count-before/count-after or state-check pattern for each tool.

### 9. Gray-Box Verification Strategy

Integration testing of complex systems requires verification beyond public interfaces. The test plan must define which gray-box techniques are used:

- **Database queries** — Which tables are queried directly to verify state changes. What columns, what conditions.
- **Container log inspection** — What log patterns indicate success or failure. What errors are never acceptable.
- **Application metrics** — Which counters, histograms, or metrics endpoints are checked. What values indicate correct operation.
- **Filesystem inspection** — What files or directories are checked inside containers.

For each test scenario that requires gray-box verification, the plan must state what is checked and why the API response alone is insufficient.

### 10. Runtime Issue Logging

The test plan must define how runtime findings are tracked in GitHub Issues:

- **Hard failures** — Assertions that fail the test. The system did not do what it should. These are filed as GitHub Issues with appropriate labels (bug, severity).
- **Warnings** — Non-fatal findings that should be reviewed. Degraded behavior, unexpected but non-breaking responses, quality concerns. Filed as GitHub Issues with a `warning` or `investigation` label.
- **Performance anomalies** — Latencies, queue depths, or throughput numbers outside expected ranges. Filed as GitHub Issues with a `performance` label.

Define the severity thresholds: what makes a finding a warning vs a bug? What quality rating from an LLM judge is acceptable vs poor?

All findings go into GitHub Issues on the project repository — not into local JSON files or markdown logs. GitHub Issues provide traceability, assignability, cross-referencing, and persistence across sessions.

### 11. Hot-Reload Testing

If the system supports runtime code deployment or configuration hot-reload, the test plan must include scenarios for:

- Verifying that runtime code deployment installs a package and the new code takes effect without restart. (For agentic services and pluggable agents, see STD-006.)
- Verifying that configuration file changes (models, prompts, tuning parameters) are picked up on the next operation.

### 12. UI Testing

If the system has a browser-based UI, the test plan must include automated browser testing. Manual visual inspection is not a substitute for automated verification. Systems without a UI skip this section.

**12.1 UI-first systems.** For systems where the UI IS the product (CLI workbenches, dashboards, control panels), UI tests are the primary acceptance criteria — not backend API tests. Backend tests verify plumbing. UI tests verify the product works. The engineering gate must not pass on backend tests alone if the UI is broken.

**12.2 Multi-model UI audit.** Multiple independent models each enumerate every clickable element, every input field, every state transition, every edge case, every race condition, and every failure recovery scenario. The master UI test list is the aggregation of all models' findings. This typically produces 5–10x more scenarios than a single model writing test cases from the requirements.

**12.3 Real-world usability testing.** Beyond automated element-by-element testing, the plan must include end-to-end usability tests: use the system to accomplish real tasks through the UI as a user would. Natural prompts, not scripted steps. If the task fails, the system has a bug — not the test. This proves the system works for its intended use case, not just that individual elements function in isolation.

### 12a. Context Stress Testing

For systems that manage LLM context windows (CLI wrappers, chat interfaces, agent orchestrators), the test plan must include progressive stress testing of context management.

**12a.1 Progressive fill.** Fill the context to each threshold defined by the system's context management strategy (e.g., 65%, 75%, 85%, 90%). Verify that each threshold triggers the expected behavior (nudges, warnings, auto-compaction).

**12a.2 Pipeline stage verification.** Context management is a multi-stage pipeline. Each stage must be verified independently — not just "did compaction happen" but "did monitoring detect the threshold, did the nudge file get created, did it get injected into the session, did the CLI receive it."

**12a.3 Multiple cycles.** The stress test must trigger the full context management cycle at least 3 times (fill → process → fill → process → fill → process). Single-cycle tests miss state accumulation bugs — counters not resetting, nudge flags not clearing, stale state from previous cycles.

**12a.4 Cold recall.** After each processing cycle, verify the system retained critical context from before processing. Topic pivots during filling create semantic islands that stress the processing algorithm at topic boundaries.

### 12b. Fresh-Container Testing

The full test suite must begin by tearing down the container and rebuilding from scratch. Testing against a container with accumulated state misses ephemerality bugs:

- Onboarding/setup that only runs on first boot
- Permissions that are correct after `chown` but wrong on fresh mount
- Database migrations that work on existing DBs but fail on fresh ones
- Symlinks or temp files that accumulate and mask path resolution bugs
- Configuration state that persists in memory but not on disk

The test startup sequence is: stop container → rebuild image → start fresh → verify health → run tests. If any test depends on state from a previous run, that dependency must be explicit in the test infrastructure, not implicit from a long-running container.

### 13. Failure Investigation Strategy

The test plan must define how failures are investigated when they occur. This is not optional — a test suite that finds bugs but doesn't guide their resolution is incomplete.

**13.1 Root cause before fix.** The plan must require that every failure is traced to its root cause before any code is changed. Reading the error message is not root cause analysis. Root cause means: identifying the exact code path that produced the failure, understanding why that code path executed, and proving the cause by reproducing it or tracing it in logs/data. Do not guess at causes. Do not pattern-match on symptoms. Read the logs, trace the data, prove the cause is real.

**13.2 Full analysis before action.** The plan must require that the investigation examines the actual error, the actual data, and the actual code path — not a summary or assumption. If a test returns a 500 error, the investigation reads the server logs to find the exception, traces the code to understand why it was raised, and verifies that the proposed fix addresses that specific exception.

**13.3 Second opinion on root cause.** Before committing to a root cause diagnosis for non-trivial bugs, consult a second LLM (via CLI or API) with the evidence gathered. Present the failing test, the error output, the relevant code, and the proposed root cause. The second model may spot alternative explanations, challenge assumptions, or identify a deeper issue. This is especially valuable when the bug is in an area the investigating model hasn't read deeply or when the first diagnosis seems too simple for the symptoms.

**13.4 Never weaken tests to make them pass.** If a test fails, the code under test has a bug. The fix goes in the application code, not the test code. Lowering thresholds, broadening assertions, adding skips, switching to softer keywords, or reclassifying a test as "flaky" are not fixes. The test is the specification — if the system doesn't meet it, the system is wrong.

**13.5 No hacks that compromise the system.** A fix must be precise and correct. A regex that matches common English words to work around an error classification problem is introducing a new bug, not fixing the old one. The fix must address the actual root cause without creating false positives, silent data loss, or behavioral changes to unrelated functionality.

**13.6 Every fix must be verified.** Deploy the fix, run the failing test, confirm the failure is gone. Then run the full suite to confirm nothing else broke. An unverified fix is not a fix.

### 14. What Is Not Tested

Explicit acknowledgment of what the test plan does not cover and why. Untested areas should be deliberate decisions, not oversights. Every NONE entry in the capability audit (§4.1) must be justified here.

**14.1 Distinguish blockers from exclusions.** An untested area is either a blocker (something that should be tested but can't be yet due to a specific dependency) or an exclusion (something deliberately not tested because it's out of scope). Blockers must reference the issue or dependency that blocks them and must be resolved — they are not permanent. Exclusions must justify why they are genuinely out of scope. "Known issue — document and move on" is not an acceptable category. If it should be tested, it's a blocker to fix.

### 15. Traceability Matrix

A table mapping every test scenario in the plan to its implementation status and execution results. This is the single source of truth for test coverage. Each row must contain:

- **Scenario ID** — Unique identifier matching the test case in section 5
- **Test file/function** — Where the test code lives (empty if not yet written)
- **Layer** — Mock or Live
- **Implementation status** — Not started / Written / Needs update
- **Last run date** — When the test was last executed
- **Result** — Pass / Fail / Error / Not run
- **Notes** — Failure details, blockers, or dependencies

The traceability matrix must be updated every time tests are written, modified, or executed. A test plan with scenarios that have no corresponding test code and no explanation is incomplete. A test that was written but never run against the actual system is unverified.

### 16. Test Suite Structure

The test plan must define the directory layout and file mapping for the test code deliverable. Each test scenario in §5 must be assigned to a specific test file. The structure must be concrete enough that multiple independent authors producing test code will create the same files in the same directories.

At minimum, specify:

- Directory structure (e.g., `tests/mock/`, `tests/live/`, `tests/browser/`)
- File names for each test module, mapped to the components they test
- Shared helper/fixture file locations
- Where test data and fixtures live

The test code deliverable (STD-005) must follow this structure exactly.

### 17. Implementation Priority

The test plan must define the order in which test modules should be implemented. Not all tests are equally important or equally foundational — some tests depend on infrastructure that other tests verify.

Specify a numbered priority list from most foundational to most complex. Foundation tests (server startup, database, config) come first. Complex integration tests (pipelines, stress tests, non-deterministic quality) come last. The priority order guides both genesis (what to write first) and review (what to verify first).

### 18. Test Execution Gates

The test plan must define what must pass before code is deployed to production. Do not prescribe the automation mechanism (CI/CD, manual, etc.) — define only the gates and their ordering. Default to manual execution unless project requirements state otherwise.

The standard gate flow:

- **Gate 1 (pre-deploy):** Mock tests pass locally. No running container needed. This gate blocks deployment to the test environment.
- **Gate 2 (deploy-test cycle):** Deploy to test container. Run live and browser tests against it. Fix issues, redeploy, rerun. This is an iterative cycle, not a single pass. This gate blocks promotion to production.
- **Gate 3 (promotion):** Same code deployed to production environment. No new testing — the code was validated in Gate 2.

For each gate, specify:
- Which test suites run
- What constitutes a pass (all green, or specific exceptions allowed?)
- What blocks progression to the next gate

### 19. Risk-Based Focus Areas

The test plan must identify the highest-risk areas of the system and ensure they receive proportionally more testing effort. Not all code paths carry equal risk — a bug in session lifecycle management is more impactful than a bug in a settings endpoint.

For each risk area, state:
- Why it is high risk (complexity, state management, user impact, failure history)
- Which test scenarios specifically address it
- What additional verification (stress testing, multi-cycle, gray-box) is applied beyond standard coverage

### 20. Final Acceptance Criteria

The test plan must define when the test suite is complete. Without explicit acceptance criteria, review rounds continue indefinitely. This is the definition of done for the test code deliverable (STD-005).

At minimum, the acceptance criteria must include:
- Post-code audit completed and all discovered scenarios implemented
- Traceability matrix fully populated with test files and results
- Zero NONE entries in the coverage audit (or explicitly justified exclusions in §14)
- All execution gates passing (§18)
- No skipped tests in the gating suite
- Structural coverage meets the threshold defined in §2.6
- Independent review of test code completed

---

## Abstraction Level

- Describe test scenarios, not test code.
- Name the behavior being tested, not the function or method.
- Specify inputs and expected outcomes, not assertions or framework syntax.
- The test plan is a specification for the test author, not the test itself.

---

## Review Criteria

When reviewing a test plan, check:

- **Discovery phase:** Was the capability audit produced by multiple independent models? Is the master list comprehensive? Was a post-code audit conducted after the code was written? Does the master list reflect both pre-code and post-code findings?
- **Requirements compliance:** Does the test plan verify compliance with the applicable engineering requirements as a prerequisite gate?
- **Two-layer strategy:** Does the plan define both mock and live test layers with clear boundaries?
- **Test infrastructure:** Is the isolation strategy defined? Configuration? Data loading? Stack lifecycle?
- **Capability audit:** Is there a complete enumeration of capabilities with coverage status?
- **Non-determinism handling:** For LLM/agent systems, does the plan define behavioral assertions and quality evaluation methods?
- **Pipeline verification:** For systems with background processing, does the plan define stage-by-stage verification, completion detection, and performance measurement?
- **Gray-box strategy:** Does the plan define what database, log, metrics, and filesystem checks are needed beyond API responses?
- **Issue logging:** Does the plan define severity categories and thresholds for runtime findings?
- **Completeness:** Are happy path and error conditions covered for each component?
- **Investigation strategy:** Does the plan require root cause analysis, second opinions, and verified fixes? Does it prohibit weakening tests?
- **UI-first priority:** For UI-based systems, does the plan make UI tests the primary gate criteria, not just backend API tests?
- **Fresh-container testing:** Does the full suite start from a torn-down, rebuilt container?
- **Stress testing:** For context-managing systems, does the plan include progressive fill, multiple processing cycles, and cold recall verification?
- **Blocker tracking:** Does "What Is Not Tested" distinguish blockers (to fix) from exclusions (genuinely out of scope)?
- **Honesty:** Are untested areas acknowledged and justified?
- **Practicality:** Can these tests actually be run? Are the dependencies realistic?
- **Test suite structure:** Does the plan define concrete file names, directories, and scenario-to-file mapping?
- **Implementation priority:** Is there a numbered priority order from foundational to complex?
- **Execution gates:** Does the plan define pre-deploy, deploy-test, and promotion gates without prescribing CI automation?
- **Risk focus:** Are high-risk areas identified with proportional testing effort?
- **Acceptance criteria:** Is there a clear definition of done for the test suite?
