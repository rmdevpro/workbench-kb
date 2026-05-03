# STD-004: Code Work Product Standard

**Purpose:** Define what a code deliverable must look like structurally, so that quorum participants producing code and reviewers evaluating it have a shared understanding of the deliverable.

---

## What This Covers

This standard defines the structural expectations for code produced by the quorum process. It does not duplicate the technical rules already defined in the applicable engineering requirements — those govern how code is written. This standard governs what the code deliverable looks like as an artifact.

## Relationship to Other Requirements

The applicable engineering requirements define how code is written — architectural patterns, async correctness, formatting, linting, testing, security, logging. This standard does not repeat those. Code must comply with the applicable engineering requirements: REQ-001 for all projects, plus REQ-002 for agentic services and REQ-003 for pluggable agents. For agentic service and pluggable agent work product additions, see STD-006.

**Test plan** (STD-003) — The test plan is produced before the code, not after. The code quorum receives the test plan as part of its anchor context alongside the functional requirements and HLD. Code is written to satisfy the test plan. This ensures the deliverable is testable by design, not tested as an afterthought.

**Test code** (STD-005) — Test code is a separate deliverable produced after the application code. The code deliverable does NOT include test code. Do not produce test files, test fixtures, or test utilities as part of this deliverable.

## Deliverable Structure

A code deliverable is a complete, buildable, runnable artifact. It contains:

1. **Application code** — The application logic, organized by function, not by layer. Every file must be complete — every function fully implemented, no stubs, no placeholders, no simplifications.

2. **Configuration templates** — Example configuration files showing all configurable options with sensible defaults and comments.

3. **Data schema** — Initial schema or migration scripts for any backing services, if applicable.

4. **Deployment definitions** — Container definitions and orchestration configuration for the full deployment, if applicable.

5. **Dependency specifications** — Dependency manifests with pinned versions for all languages used.

6. **README** — What the system is, how to deploy it, how to configure it, how to use it.

7. **Package registration** (if applicable) — Registration in the project's package manager following the applicable conventions. For agentic services and pluggable agents using AE/TE packages, see STD-006.

## Output Format

The deliverable is a sequence of files. Each file is presented as:

- A file path header (the full relative path)
- Followed by the complete file contents in a fenced code block

No other content. No introductory paragraphs, no explanatory commentary between files, no closing summaries, no suggestions for next steps. The output is files only.

## Completeness Requirements

- **Every feature specified in the functional requirements and designed in the HLD must be fully implemented.** Do not simplify, stub, or declare any functionality as "no-op," "placeholder," or "to be implemented." If the requirements specify it, the code implements it.
- **Every file must be complete.** Do not abbreviate with comments like "... and many other functions," "similar to above," or "rest of implementation here." Write every function, every method, every line.
- **This is a single, final deliverable.** Do not offer to continue, produce additional versions, or suggest next steps. There is no follow-up. Everything must be in this response.

## What the Code Deliverable Is NOT

- It is not a design document. If the code needs explaining, the code is unclear.
- It is not a prototype, sketch, or "version 1." The deliverable must be buildable and runnable as delivered.
- It does not re-debate architectural decisions settled in the HLD. The HLD made those choices. The code implements them.
- It does not include test code. Test code is a separate deliverable (STD-005).
- It does not include conversational text, meta-commentary, or discussion. Output is code files only.

## Review Criteria

When reviewing a code deliverable, check:

- **Requirements compliance:** Does the code satisfy the functional requirements and the applicable engineering requirements?
- **HLD compliance:** Does the code implement the architecture described in the HLD?
- **Concept fidelity:** Does the code faithfully represent the domain model and design rationale?
- **Test plan alignment:** Is the code testable against the test plan? Does it satisfy the scenarios defined there?
- **Completeness:** Could a developer clone this, follow the README, and have a running system? Is every feature from the requirements implemented — not stubbed, simplified, or deferred?
- **Correctness:** Does the code do what it claims? Are there obvious bugs or logic errors?
- **Clarity:** Is the code readable and maintainable? Could another developer understand and modify it?

## What NOT to Review

- Do not review for style preferences beyond what the engineering requirements mandate.
- Do not suggest alternative architectural approaches — those were settled in the HLD.
- Do not request additional features beyond what the functional requirements specify.
