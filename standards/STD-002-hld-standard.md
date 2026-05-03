# STD-002: High-Level Design Standard

**Purpose:** Define what a High-Level Design document must contain and at what level of abstraction, so that quorum participants and reviewers have a shared understanding of the deliverable.

---

## What an HLD Is

An HLD describes the architecture of a system at a level above implementation detail. It answers: what are the major components, how do they relate, what technology choices were made and why, how does data flow, what are the interfaces, and how is it deployed.

An HLD gives an implementer enough architectural direction to start building. The implementer makes the detailed design decisions — the HLD points them in the right direction.

## What an HLD Is NOT

An HLD does not include:

- Class designs or function signatures
- Full database DDL or schema definitions (table names and roles yes; column types and indexes no)
- Code examples or snippets
- Container configuration files or reverse proxy configuration
- Error handling logic or retry policies
- Specific library API usage (e.g., which class parameters to set, which function to call)
- Configuration file syntax beyond illustrative examples

These belong either in the code (which serves as its own detailed design) or in implementation documentation.

## Required Sections

1. **System Overview** — What the system is, what problem it solves, and how it relates to the domain model or foundational concepts that motivate the design.

2. **Component Architecture** — Major components, their roles, how they connect. Include a diagram. For containerized systems: which containers, which images, network topology, volume mounts. For packages: which modules, their responsibilities, how they compose.

3. **Data Flow** — How data moves through the system end-to-end. A numbered sequence from input to output covering the critical path.

4. **Technology Choices** — What technologies are used and why. State the decision and the rationale, not the API details.

5. **Interface Design** — What interfaces the system exposes (protocols, endpoints, tool inventories) at a contract level. Not input/output schemas — just what exists and what it does.

6. **Configuration Model** — How the system is configured, what's configurable, what the configuration categories are.

7. **Resilience and Operational Model** — How the system handles failure, what's eventually consistent, what the degradation model is, how it starts up.

8. **Known Constraints and Trade-offs** — Honest acknowledgment of the limitations inherent in the design choices.

## Abstraction Level

- Name components and their responsibilities, not their internal implementation.
- Describe data flow as a sequence of what happens, not how each step is coded.
- Name technology choices at the platform level (e.g., "Redis for job queues") without specifying specific commands, API calls, or library parameters.
- When requirements mandate specific patterns (e.g., "all logic in state machines" or "all routes in Express"), the HLD should describe which flows exist and what they do — not their internal implementation details.
- Diagrams should show component relationships, not internal structure.

## Scope Drift

HLD reviews tend to drift into implementation territory over successive rounds. The general principle: if an issue is about a specific technology API, command, parameter, or processing semantic rather than about component relationships, data flow, or architectural decisions, it is below HLD level.

Signs of scope drift that the PM or reviewers should flag:
- Suggesting specific database commands or query syntax
- Requesting specific library class parameters or configuration
- Requesting API parameter designs for individual tools
- Requesting database column types or index definitions
- Requesting processing semantics (e.g., deduplication behavior, job ordering)
- Requesting protocol-level details already specified in the functional requirements (e.g., HTTP status codes)

These are all legitimate implementation concerns — they just don't belong in an HLD.

## Review Criteria

When reviewing an HLD, check:

- **Requirements compliance:** Does the HLD satisfy the applicable engineering requirements and the functional requirements it was built from?
- **Concept fidelity:** Does the HLD faithfully represent why the system is designed this way — the domain model, foundational concepts, and design rationale that motivate the architecture?
- **Architectural soundness:** Are component boundaries clean? Do data flows make sense? Are there circular dependencies or unnecessary coupling?
- **Best practice:** Do the technology choices follow established patterns for the technologies selected?
- **Completeness:** Does the HLD cover the sections above at the right level of detail?
- **Clarity:** Could a developer read this and understand the architecture without asking clarifying questions?
- **Scope:** Is every element in the document at HLD level? Flag anything that belongs in implementation.
