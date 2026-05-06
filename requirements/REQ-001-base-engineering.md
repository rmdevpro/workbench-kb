# REQ-001: Base Engineering Requirements

These requirements apply to **every project** in the ecosystem, regardless of language or role: agentic services, pluggable agents, supporting tools (e.g. the Agentic Workbench), CLIs, glue scripts, anything that gets deployed.

The rules here are language-neutral. Agentic-service-specific rules (LangGraph mandatory, AE/TE composition, MCP, Prometheus) live in `REQ-002-agentic-service.md`. Pluggable-agent-specific rules (package structure, hot install, statelessness) live in `REQ-003-pluggable-agent.md`.

## Compliance

Every numbered item below is a hard requirement. A project must satisfy every applicable item. Failure to satisfy a requirement is non-compliance.

An exception may be requested when a requirement genuinely cannot or should not be applied to a specific case. **Only the user may approve an exception.** An agent — whether an Imperator, a code-writing assistant, or any other automated actor — must not self-approve, infer, or assume an exception. An agent that encounters a requirement it believes should not apply must surface the conflict to the user and wait for an explicit approval; it must not proceed by silently relaxing the requirement.

A requirement that has not been explicitly excepted by the user remains binding. An exception that has not been recorded by the user with explicit approval does not exist.

Approved exceptions must be recorded in the exception registry (`exception-registry.md`) with: the requirement being excepted, the specific scope of the exception, the justification, and the user approval. The exception registry is the single source of truth for what is allowed to deviate from these requirements; the same model and approval process applies to `REQ-002`, `REQ-003`, and any other requirements documents in this set.

---

## 1. Code Quality

**1.1 Code clarity**

- Code must be clear, readable, and maintainable.
- Variables, functions, and classes must have descriptive names.
- Functions must be small and focused — each must do one thing well.
- Comments must explain *why*, not *what*.

**1.2 Code formatting**

- Code must be formatted with the standard formatter for its language (Python: `black`; JavaScript / TypeScript: project-defined; Go: `gofmt`).
- The formatter check must pass without errors before code lands.

**1.3 Code linting**

- Code must pass the standard linter for its language without errors (Python: `ruff`; JavaScript / TypeScript: `eslint` or equivalent).

**1.4 Unit testing**

- All programmatic logic must have corresponding tests covering the primary success path and common error conditions.

**1.5 Version pinning**

- All dependencies must be locked to exact versions.
- Python: `==` in `requirements.txt`. Node: exact versions in `package.json`. Go: pinned via `go.mod` + `go.sum`.
- The most recent stable version must be used unless a specific exception applies.

---

## 2. Security

**2.1 No hardcoded secrets**

- Credentials must not appear in code, Dockerfiles, compose files, or any committed file.
- Credentials must load from environment variables or credential files at runtime.
- Repositories must ship example/template credential files; real credentials must be gitignored.

**2.2 Input validation**

- All data from external sources (user inputs, API responses, tool inputs) must be validated before use.

**2.3 Null / undefined checking**

- Variables that may be `None`, `null`, or `undefined` must be explicitly checked before attribute access.

---

## 3. Logging and Observability

**3.1 Logs to stdout / stderr**

- Normal logs must go to stdout, errors to stderr. Log files inside containers or processes must not be created.

**3.2 Structured logging**

- Logs must be JSON, one object per line.
- Each log line must include: timestamp (ISO 8601), level (`DEBUG` / `INFO` / `WARN` / `ERROR`), message, plus context fields appropriate to the event.

**3.3 Log levels**

- Levels `DEBUG`, `INFO`, `WARN`, `ERROR` must be supported. Default level must be `INFO`. The level must be configurable.

**3.4 Log content**

- Lifecycle events, errors with context, and performance metrics must be logged.
- Secrets, full request / response bodies, and health-check successes (only state changes) must not be logged.

**3.5 Specific exception handling**

- Blanket catch-all exception handlers must not be used. Caught exceptions must be specific and anticipated.

**3.6 Resource management**

- All external resources (file handles, database connections, network sockets) must be reliably closed via context managers, `try`/`finally`, or equivalent.

**3.7 Error context**

- Logged errors and raised exceptions must include enough context to debug.

**3.8 Pipeline observability**

- Multi-stage processing pipelines must support a verbose logging mode that reports what happens at each stage, including intermediate outputs and per-stage timings.
- Verbose mode must be togglable via configuration. It must not require a code change to enable, and it must not be always on.

---

## 4. Async Correctness

**4.1 No blocking I/O in async functions**

- Async functions must use async libraries appropriate to the language / runtime.
- Synchronous blocking calls in async context are forbidden.

---

## 5. Communication

**5.1 Health endpoint**

- Systems that expose HTTP must provide `GET /health` returning `200` when healthy and `503` when unhealthy, with per-dependency status in the body.

---

## 6. Resilience

**6.1 Graceful degradation**

- Failure of an optional component must cause degraded operation, not a crash.
- Core operations must continue with reduced capability when an optional component fails.
- The health endpoint must report degraded status under partial failure.

**6.2 Independent startup**

- Components must start and bind ports without waiting for dependencies.
- Dependency unavailability must be handled at request time, not at startup.

**6.3 Idempotency**

- Operations that may be retried must be safe to execute more than once with the same input — no unintended side effects.

**6.4 Fail fast**

- On invalid configuration, missing required dependencies, or corrupt state, the system must fail immediately with a clear error. It must not proceed and produce wrong results silently.

---

## 7. Configuration

**7.1 Configurable external dependencies**

- Inference providers, model selection, and similar external dependencies must be configurable via config file or environment variable. They must not be hardcoded.

**7.2 Externalised configuration**

- Configuration, parameters, and content that may change between deployments or over time must be externalised from application code.
- Externalised values include: prompt templates, model parameters, retry counts, timeouts, thresholds, file paths, URL patterns, and any value someone might reasonably want to change without changing code.

**7.3 Hot-reload vs startup configuration**

- Runtime-changeable settings (models, tuning parameters) must be read per operation, with no restart required.
- Infrastructure settings (database connections, ports) must be read at startup; a restart is required to change them.

---

## 8. Deployment

**8.1 Compose self-sufficiency**

- `docker compose up -d --build` must produce a fully working deployment with no wrapper scripts, manual steps, or external tooling.
- All init logic, dependency ordering, healthchecks, and permissions must be handled by the Dockerfile, entrypoint, and compose file.
- Wrapper scripts (`deploy.sh`, `start.sh`) must not be used to work around gaps in compose. If compose alone doesn't work, the compose file and Dockerfile must be fixed.

**8.2 Compose override files**

- Host-specific or environment-specific values must be applied via a `docker-compose.override.yml` file. This is the standard Docker Compose mechanism and is not a violation of §8.1.
- The shipped `docker-compose.yml` in the service repository must remain generic and portable — anyone who clones the repo must be able to run `docker compose up --build` and get a working stack.
- Per-host overrides must live in the Admin repo at `compose/<host>/<service>/docker-compose.override.yml` and merge at deploy time. See `infrastructure/INF-003-container-deployment.md` for the override pattern.
- Overrides may set: image references (e.g. private registry), bind-mount host paths, host-specific ports, internal hostnames, environment-variable values. Overrides must not introduce new application logic or mask defects in the shipped compose file.

**8.3 Environment separation**

- Environment-specific values (ports, volume paths, container names, env vars) must be the only difference between deployment targets, and they must be applied through override files (§8.2).
- Application code, Dockerfile, and entrypoint must be identical across environments.

---

## Document Map

| Topic | See |
|---|---|
| Agentic-service requirements (LangGraph, AE/TE, MCP, Prometheus, container architecture) | `REQ-002-agentic-service.md` |
| Pluggable-agent requirements (package structure, hot install, host contract) | `REQ-003-pluggable-agent.md` |
| Architecture concepts | `architecture/ARC-001-platform-overview.md` and onward |
| Document, code, and test standards | `standards/STD-001` through `STD-007` |
| Container deployment model | `infrastructure/INF-003-container-deployment.md` |
