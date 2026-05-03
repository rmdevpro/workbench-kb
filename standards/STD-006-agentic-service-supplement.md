# STD-006: Agentic Service and Pluggable Agent Supplement to Work Product Standards

**Purpose:** Define the additions to the base work product standards (STD-001 through STD-005) that apply to Agentic Services and Pluggable Agents. Projects of these types reference both the base standards and this supplement.

---

## Applicability

This supplement applies to:
- **Agentic Services** — REQ-002 compliant
- **Pluggable Agents** — REQ-003 compliant

Standalone applications (e.g., Agentic Workbench) do NOT reference this supplement.

---

## Engineering Requirements Reference

Agentic Services must comply with:
- **REQ-001** — base engineering requirements
- **REQ-002** — agentic service requirements (architecture, StateGraph, hot-reload, MCP, logging)

Pluggable Agents must comply with:
- **REQ-001** — base engineering requirements
- **REQ-003** — pluggable agent requirements (package structure, hot install, statelessness)

---

## Additions to STD-002 (HLD Standard)

### StateGraph Documentation

When the HLD describes an Agentic Service or Pluggable Agent that uses StateGraphs, it should describe which flows exist and what they do — not their internal node names, edge definitions, or state schemas. Those are implementation details.

---

## Additions to STD-003 (Test Plan Standard)

### Engineering Gate (§1)

For Agentic Services, the engineering gate verifies compliance with REQ-001 (base) and REQ-002. For Pluggable Agents, with REQ-001 (base) and REQ-003.

### Coverage Targets (§4)

In addition to the base coverage targets:
- Every MCP tool must be tested with realistic inputs and verified outputs.
- Every Imperator tool must be tested for real side effects, not just conversational acknowledgment. "The Imperator said it wrote the file" is not verification — checking that the file exists on disk is. Define the count-before/count-after or state-check pattern for each tool.
- Every background pipeline stage must be verified (embeddings, extraction, assembly, etc.).

### Pipeline Verification (§7)

Agentic Service pipelines typically include embedding, extraction, and assembly stages. Each stage must be verified independently:
- Embeddings: verify vectors exist in the database after ingestion
- Extraction: verify extraction marks set on processed records
- Assembly: verify summary rows created

### Gray-Box Verification (§9)

Agentic Services with Prometheus metrics must include metrics scraping in their gray-box strategy:
- Which Prometheus counters and histograms are checked
- What values indicate correct operation
- Scrape `/metrics` endpoint after test runs
- Parse histogram buckets to estimate percentiles (p50, p90, p99)

### Hot-Reload Testing (§11)

Agentic Services and Pluggable Agents that use `install_stategraph` must verify:
- `install_stategraph` installs a package and the new code takes effect without container restart
- The bootstrapper discovers and loads AE/TE packages from entry_points
- Configuration hot-reload picks up changes to config files, prompt files, and credential files

---

## Additions to STD-004 (Code Standard)

### Package Registration

AE/TE packages must include entry_points registration in `pyproject.toml` following the TE package structure defined in REQ-002 §12. The entry_points group identifies the package as AE or TE and enables discovery by the bootstrap kernel.

---

## Additions to STD-005 (Test Code Standard)

### Infrastructure Helpers

Test helpers for Agentic Services and Pluggable Agents should include:
- `mcp_call(client, tool_name, args)` — MCP JSON-RPC tool invocation
- `docker_exec(container, command)` — run command inside a container
- `docker_logs(container, tail)` — get container logs
- SSH routing for remote hosts

### Prometheus Performance Reports

After test runs, scrape Prometheus `/metrics` and generate a performance report:
- Parse histogram buckets to estimate percentiles
- Generate markdown table with latencies per tool
- Include queue depths, job counts, key findings

### install_stategraph Testing

```python
def test_hot_reload_ae_package(http_client):
    resp = mcp_call(http_client, "install_stategraph", {"package_name": "my-ae"})
    assert resp.status_code == 200
    result = extract_mcp_result(resp)
    assert result["status"] == "installed"
```

---

**Related:** [REQ-001](../requirements/REQ-001-base-engineering.md) · [REQ-002](../requirements/REQ-002-agentic-service.md) · [REQ-003](../requirements/REQ-003-pluggable-agent.md)
