# REQ-003: Pluggable Agent Requirements

These requirements apply to **Pluggable Agents** — Python packages containing agentic intelligence that install at runtime into a host. The canonical host is an agentic service's TE side; the pattern is host-agnostic and applies wherever a Pluggable Agent is loaded.

These requirements are additive to `REQ-001-base-engineering.md`. The base requirements apply in full. This document defines what makes a Pluggable Agent a Pluggable Agent. For the architectural concept, see `architecture/ARC-003-pluggable-agent.md`.

## Compliance

Every numbered item below is a hard requirement. A Pluggable Agent must satisfy every applicable item. Failure to satisfy a requirement is non-compliance.

Exceptions are governed by the same rules as `REQ-001`: only the user may approve an exception, agents must not self-approve or assume one, and approved exceptions must be recorded in `exception-registry.md` with requirement, scope, justification, and user approval. See `REQ-001-base-engineering.md §Compliance` for the full exception model.

---

## 1. Package Structure

**1.1 Standard Python package**

- A Pluggable Agent must be a standard Python package with a `pyproject.toml`.
- The package must be built as a wheel.
- The package must be published to a package index (the Development Cache, or PyPI for public packages).
- The package must be installable via `pip install` with no additional build steps.

**1.2 Entry-point registration**

- The package must register itself in `pyproject.toml` against the host's TE entry-point group:

    ```toml
    [project.entry-points."<host>.te"]
    <agent-name> = "<module>.register:build_graph"
    ```

- The package must expose `build_graph(context: HostContext) -> StateGraph`.
- `build_graph` must also accept a plain `dict` argument for backward compatibility.

**1.3 Package data**

- Non-Python runtime files (prompt templates, JSON defaults, fixtures) must be declared in `pyproject.toml` under `[tool.setuptools.package-data]`.
- Files must be accessed via `Path(__file__).resolve().parent` relative paths.
- Hardcoded absolute paths are forbidden.

**1.4 LangGraph mandate**

- StateGraphs and any internal flows must be LangGraph StateGraphs.
- Procedural code outside the graph is forbidden.

---

## 2. Host Contract

(Concept: `ARC-003 §The Host Contract`.)

**2.1 Input state**

- The host must invoke the agent's StateGraph with `{"payload": <request_body>}`.
- The agent's StateGraph must resolve its initial state from the payload in its first node.
- The package must document, in its README or equivalent, every payload field it expects.

**2.2 Output state**

- The agent's StateGraph must include `final_response` in its output state.
- Tool calls and tool results that should be persisted must be included in the output messages list.

**2.3 No infrastructure assumptions**

- The agent must not own containers, volumes, or network configuration.
- Backing services must be accessed only through the host's connection pools and peer-proxy interface.
- Filesystem paths beyond those provided by the host must not be assumed.

**2.4 Inference**

- All LLM calls must go through the host-provided inference interface.
- The agent must not instantiate LLM clients directly.

**2.5 Peer access**

- Calls to other agentic services must go through the host's peer proxy.
- Direct network calls to ecosystem services are forbidden.

---

## 3. Imperator

(Pattern: `ARC-003 §The Imperator Pattern`.)

**3.1 Mandate**

- Every Pluggable Agent must contain an Imperator as its prime agent.

**3.2 Identity and Purpose**

- Identity must include the agent's domain.
- Purpose must be established within the domain.
- Identity and Purpose must be baked into the package and expressed through the system prompt template, prompts, and tool registrations.
- Identity and Purpose must not change at runtime.
- Identity and Purpose must not be derivable from calling context.

**3.3 Naming conventions**

These conventions should be followed unless the package's structure justifies a deviation:

| Element | Convention |
|---|---|
| Main flow file | `flows/imperator.py` |
| State class | `<Name>ImperatorState` |
| Graph builder | `build_imperator_graph()` |
| Entry-point registration | `register.py` exposing `build_graph` |
| System prompt templates | `prompts/` |

**3.4 ReAct**

- The Imperator must be implemented as a graph-based ReAct agent.
- The ReAct loop must be graph structure, not a procedural loop inside a single node.

---

## 4. Statelessness

**4.1 No internal state**

- The Pluggable Agent must store nothing internally between invocations.
- Conversation state must live in the platform's Context Broker (or the host-provided equivalent).
- Operational state must be owned by the host.
- Generated artifacts must go to shared storage. They must not be written to the package directory.

**4.2 Multiplicity**

- Multiple instances of the same agent must be safe to run concurrently with no coordination.
- The agent must not assume single-instance execution.

**4.3 Conversation continuity**

- The agent must not assume that successive invocations of the same conversation hit the same Python process.
- The agent must rely on host-provided context (fetched per invocation) for conversational continuity.

---

## 5. Versioning and Hot Install

**5.1 Semantic versioning**

- Pluggable Agent packages must follow semantic versioning.
- The version must be bumped on every publish.

**5.2 Hot install**

- The agent must be safe to install via the host's `install_stategraph()` without restarting the host.
- The agent must not hold global state that persists across versions.
- The agent must not have import-time side effects that modify the host.
- The agent must not assume the same Python process services successive invocations of the same conversation.

---

## 6. Configuration

**6.1 TE configuration ownership**

- TE-side configuration (Imperator settings, inference assignments, cognitive parameters, tool registrations) must be owned by the agent package.
- TE-side configuration must not be embedded in the host's infrastructure config.
- When the package is upgraded or replaced, its configuration must change with it.

**6.2 Tool registrations**

- The package must declare which tools the Imperator needs from the host.

---

## Document Map

| Topic | See |
|---|---|
| Architectural overview of a Pluggable Agent | `architecture/ARC-003-pluggable-agent.md` |
| Agentic-service requirements (the canonical host) | `REQ-002-agentic-service.md` |
| Platform overview | `architecture/ARC-001-platform-overview.md` |
| Base engineering requirements | `REQ-001-base-engineering.md` |
| Code standards | `standards/STD-004-code-standard.md` |
