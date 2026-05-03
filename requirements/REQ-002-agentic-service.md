# REQ-002: Agentic Service Requirements

These requirements apply to **agentic services** — the infrastructure services on the Blueprint Agentic platform (Context Broker, Modular Agent Host, Inference Broker, Development Cache, Observability Platform, Agentic File System, Conversation Broker), and any new agentic service built to the same pattern.

These requirements are additive to `REQ-001-base-engineering.md`. The base requirements apply in full. This document defines what makes an agentic service an agentic service. For the architectural concepts referenced here, see `architecture/ARC-002-agentic-service.md`.

## Compliance

Every numbered item below is a hard requirement. An agentic service must satisfy every applicable item. Failure to satisfy a requirement is non-compliance.

Exceptions are governed by the same rules as `REQ-001`: only the user may approve an exception, agents must not self-approve or assume one, and approved exceptions must be recorded in `exception-registry.md` with requirement, scope, justification, and user approval. See `REQ-001-base-engineering.md §Compliance` for the full exception model.

---

## 1. LangGraph Architecture

**1.1 LangGraph mandate**

- All programmatic and cognitive logic must be implemented as LangGraph StateGraphs.
- Procedural code outside the graph is forbidden.

**1.2 Graph structure**

- Each distinct operation must be a node.
- State must flow between nodes via typed state.
- Flow control must be expressed as graph edges and conditional routing.
- Procedural `if`/`else` chains inside nodes for flow control are forbidden.

**1.3 Node restrictions**

- Nodes must not contain loops.
- Nodes must not contain sequential multi-step logic.
- Nodes must not contain branching that controls what happens next.
- A node that does multiple unrelated things sequentially must be split into separate nodes.
- Branching about what happens next must be expressed as a conditional edge.
- Loops (e.g. tool-calling cycles) must be expressed as graph edges between nodes.

**1.4 Route handlers**

- The web server must initialise and invoke compiled StateGraphs only.
- Application logic in route handlers is forbidden.

**1.5 Standard components**

- LangChain and LangGraph standard components must be used where applicable: chat-model calls, embeddings, structured output, retrievers, vector stores, tool binding, retry logic, checkpointing.
- Native APIs may be used only when no standard component fits, with a documented exception.

**1.6 State immutability**

- StateGraph node functions must not modify input state in place.
- Each node must return a new dictionary containing only the state fields it updates.

**1.7 Checkpointing**

- LangGraph checkpointing must be used for long-running agent loops.
- Short-lived background flows are exempt.

---

## 2. AE and TE Composition

(Concept: `ARC-002 §Pluggable Code: AE and TE`.)

**2.1 Two-part structure**

- Pluggable code must be structured as AE and TE.
- Both AE and TE must ship as Python packages.

**2.2 Entry-point groups**

- Each agentic service must define entry-point groups `<service>.ae` and `<service>.te`.
- The kernel must scan both groups on startup and after every `install_stategraph()` call.

**2.3 Direction of invocation**

- The AE must call into the TE.
- All external traffic must enter through the AE.
- The kernel must not route external traffic directly to a TE.

**2.4 TE configuration ownership**

- TE configuration must live with the TE package.
- TE configuration must not live in the AE's infrastructure config.
- Replacing or upgrading a TE package must not require changing AE infrastructure settings.

---

## 3. Host Contract

(Concept: `ARC-002 §The Host Contract`.)

**3.1 Publication**

- The kernel must publish a stable host contract.

**3.2 Required contract items**

The kernel must provide each of the following:

- Connection pool to Postgres.
- Logging configuration.
- Metrics registry for Prometheus instrumentation.
- Peer-proxy access for calling other agentic services.
- Raw request payload to TE StateGraphs as `{"payload": <body>}`.

**3.3 Stability**

- A new kernel that satisfies the host contract must not require changes to installed AE or TE packages.
- A new pluggable package that conforms to the contract must install into any kernel satisfying the same contract.

---

## 4. Imperator

(Pattern: `ARC-003 §The Imperator Pattern`.)

**4.1 Mandate**

- Every agentic service must include an Imperator.
- The Imperator must be the front door for all conversational interaction with the service.

**4.2 Identity and Purpose**

- Identity must include the service domain.
- Purpose must be established within the domain.
- Identity and Purpose must be baked into the TE package and expressed through the system prompt template, prompts, and tool registrations.
- Identity and Purpose must not change at runtime.
- Identity and Purpose must not be derivable from calling context.

**4.3 Service operation**

- The Imperator must operate its service: handling prose goals, dispatching the service's own tools, and decomposing multi-step goals.

**4.4 Metacoding**

- The Imperator must be the path through which the service's own configuration, code, or recipes are modified by a person or another agent.

**4.5 Autonomy**

- Designs must keep full Autonomy as a long-term direction.
- Designs must not regress this trajectory.

**4.6 ReAct**

- The Imperator must be implemented as a graph-based ReAct agent.
- The ReAct loop must be graph structure, not a procedural loop inside a single node.

---

## 5. Standard Service Stack

(Architectural roles: `ARC-002 §Standard Service Stack`.)

**5.1 Mandatory containers**

Every agentic service must ship every container in this table:

| Container | Image | Required |
|---|---|---|
| Gateway | `nginx:<pinned>-alpine` | Yes |
| Langgraph | Custom (Python + LangGraph) | Yes |
| Postgres + pgvector | `pgvector/pgvector:pg16` | Yes |
| Log shipper | Custom sidecar | Yes |
| Dkron | `dkron/dkron:4` | Yes |
| UI | Custom (Gradio) | Yes |
| Alerter | Custom sidecar | Yes |
| Other purpose-specific | (as needed) | As needed |

**5.2 Off-the-shelf where possible**

- Custom-built containers in the standard stack must be limited to: Langgraph, log shipper, UI, alerter.
- Building a custom container for a role with an established off-the-shelf image requires a documented exception.

**5.3 Image pinning**

- All images must be pinned to specific tags.

**5.4 Mandatory role assignments**

- Postgres + pgvector must be the relational + vector store and must hold the `system_logs` table.
- The log shipper must tail private-network containers via the Docker API and must write to `system_logs`.
- Dkron must be the autoprompter.
- The UI must speak the standard OpenAI-compatible endpoint and must work against any agentic service.
- The alerter must receive CloudEvents webhooks from Imperator notifications and must fan them out to external channels.

---

## 6. Container Construction

**6.1 Root usage**

- Root privileges must be used only for system-package installation and user creation.
- The `USER` directive must immediately follow user creation.
- Application code, file operations, and runtime tasks must not run as root.

**6.2 Service account**

- Each custom container must run as a dedicated non-root user.
- UID and GID must be defined in the Dockerfile.
- UID and GID must be consistent across the service's container group.

**6.3 File ownership**

- File ownership must be set with `COPY --chown` rather than `chown -R`.

**6.4 Dockerfile HEALTHCHECK**

- Every container's Dockerfile must include a `HEALTHCHECK` directive.
- `HEALTHCHECK` must use `curl` or `wget` against the container's health endpoint.

---

## 7. Network Topology

(Concept: `ARC-002 §Network Topology`.)

**7.1 Two-network pattern**

- Each agentic service must use a public network and a private per-service bridge (`<service>-net`).

**7.2 Network membership**

- Only the gateway may join the public network.
- The Langgraph container, Postgres, sidecars, and any other internal containers must join only the private network.

**7.3 Gateway as inbound boundary**

- The gateway must handle all inbound cross-network traffic and proxy it to the kernel on the private network.
- The Langgraph container must not join the public network.
- Outbound traffic from the kernel goes directly via Docker's bridge networking; it does not pass through the gateway.
- Peer service names must be stable inbound addresses regardless of internal container changes.

**7.4 Service-name DNS**

- Inter-container communication must use Docker Compose service names.
- IP addresses must not be used for inter-container communication.

---

## 8. Storage

**8.1 Volume pattern**

- Containers must use bind mounts for persistent data.
- Configuration and credentials must mount separately from system-generated data.
- Paths inside containers must be fixed.
- Host paths must be controlled by the compose file or per-host override.

**8.2 Database storage**

- Each backing service must use its own data subdirectory.
- Backing-service containers must use bind mounts at their declared `VOLUME` paths.

**8.3 Backup and recovery**

- Persistent state must live under a single host directory per service (`/srv/<service>/`).
- That directory must be the unit of backup.
- Schema changes between versions must be versioned and must be applied automatically on startup.
- Migrations must be forward-only and non-destructive.

**8.4 Credential management**

- Credentials must live in a dedicated location and must load via `env_file` in compose.
- Application code must read credentials from environment variables.
- Repositories must ship example credential files.
- Real credentials must be gitignored.

---

## 9. Deployment

**9.1 Compose**

- An agentic service must ship as a standalone repository with its own `docker-compose.yml`.
- The compose file must define all services, networks, and volumes for that service.
- Host-specific values must be applied via `docker-compose.override.yml` in the Admin repo.
- The shipped compose file must not be modified for host-specific concerns.

**9.2 Zero-script deployment**

- An agentic service must be fully operational after `git clone` followed by `docker compose up`.
- No deployment scripts, manual directory creation, `chmod`, or other prerequisite steps may be required beyond providing credentials.
- Filesystem needs (directories, permissions, volume initialisation) must be handled by the Dockerfile, compose volume definitions, entrypoint scripts, and named volumes.

**9.3 Health-check architecture**

- Each agentic service must implement two health-check layers: Docker `HEALTHCHECK` per container and HTTP `/health` endpoint.
- The Langgraph container must perform the dependency checks.
- The gateway must proxy the health response.

**9.4 Eventual consistency**

- Systems with multiple datastores must be eventually consistent, not transactionally consistent.
- The primary datastore (typically Postgres) must be the source of truth.
- Failed background jobs must retry with backoff.
- Data must not be lost due to downstream processing failure.

---

## 10. Interfaces

**10.1 MCP**

- The gateway must expose `/mcp` for MCP tool access via HTTP/SSE.
- MCP tools must follow the naming convention `<domain>_<action>`.
- MCP tool handlers must be AE-side compiled StateGraphs invoked by thin route handlers.

**10.2 OpenAI-compatible chat**

- Every agentic service must expose `/v1/chat/completions` (OpenAI-compatible).
- Services hosting pluggable agents must accept the hosted agent's name in the `model` field.

**10.3 Authentication**

- For shared networks, authentication must be applied at the gateway without modifying the application.

---

## 11. Prometheus Metrics

**11.1 Endpoints**

- Every agentic service must expose `GET /metrics` in Prometheus exposition format.
- Every agentic service must expose `POST /metrics_get` returning the same exposition text in a JSON envelope.

**11.2 Standard metrics**

Every agentic service must emit at least:

| Metric | Type | Labels |
|---|---|---|
| `mcp_requests_total` | Counter | `service`, `tool`, `status` |
| `mcp_request_duration_seconds` | Histogram | `service`, `tool` |
| `mcp_requests_errors_total` | Counter | `service`, `tool`, `error_type` |

**11.3 Implementation**

- Metrics must be produced inside StateGraphs.
- Route handlers for `/metrics` and `/metrics_get` must be thin wrappers invoking a compiled StateGraph.

---

## 12. Dynamic Loading

**12.1 Runtime installation**

- Both AE and TE packages must be installable at runtime without restarting the Langgraph container.
- The mechanism must be `install_stategraph(<package_name>)`.

**12.2 Package discovery**

- StateGraph packages must be discovered via Python `setuptools` entry points.
- The kernel must scan both the `.ae` and `.te` entry-point groups on startup and after every `install_stategraph()` call.

**12.3 Package publishing**

- StateGraph packages must be standard Python packages with semantic versioning.
- Packages must be published to a package index before installation.

---

## Document Map

| Topic | See |
|---|---|
| Architectural overview of an agentic service | `architecture/ARC-002-agentic-service.md` |
| Platform overview | `architecture/ARC-001-platform-overview.md` |
| Pluggable-agent requirements (TE-side packages) | `REQ-003-pluggable-agent.md` |
| Base engineering requirements | `REQ-001-base-engineering.md` |
| Container deployment model | `infrastructure/INF-003-container-deployment.md` |
| Storage conventions | `infrastructure/INF-002-storage-conventions.md` |
| Deployment runbook | `runbooks/RUN-001-deployment.md` |
