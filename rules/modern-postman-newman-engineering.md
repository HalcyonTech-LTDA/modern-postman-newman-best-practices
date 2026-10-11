---
description: Automatically enforces modern Postman and Newman automated API testing standards: collection craftsmanship and v2.1 modularity, modern pm.* Chai BDD assertions, defensive parsing and schema validation, zero-secret environment decoupling, dynamic request chaining and workflow control, containerized orchestration with official Docker runners, deterministic CI/CD quality gates, and mandatory living README.md runbook documentation discipline.
trigger: model_decision
---

# Modern Postman & Newman Engineering Guardrails & Skill Router

When authoring, reviewing, refactoring, or executing Postman collections, environments, pre-request/test scripts, Newman command-line executions, Docker container runners, CI/CD automated test pipelines, or API test documentation, strictly adhere to Modern Postman & Newman Engineering Standards and the corresponding specialized skills:

| When working on... | Consult & Activate Skill | Key Focus Areas |
| :--- | :--- | :--- |
| **Collection Design & Script Craftsmanship** | `modern-postman-collection-craftsmanship` | Collection v2.1 format, modular folder hierarchies, modern `pm.*` API, Chai BDD assertions, defensive response parsing, JSON Schema contract validation, dynamic synthetic data, and loop-safe workflow chaining (`postman.setNextRequest`). |
| **Variable Scoping, Secrets & Data-Driven Fixtures** | `modern-postman-environments-and-secrets` | Five-tier variable scope hierarchy, zero hardcoded secrets in source control, environment template contracts, runtime CLI secret injection (`--env-var`), data-driven testing (`--iteration-data`), and variable sanitation. |
| **Living Documentation & Runbooks** | `modern-newman-living-documentation` | Mandatory dedicated `README.md` runbook, System Under Test (SUT) scope, scenario & coverage matrices, dynamic environment contracts, CLI/Taskfile/Docker execution recipes, and atomic PR synchronization checklists. |
| **Container Orchestration & CI/CD Quality Gates** | `modern-newman-orchestration-and-ci` | Official pinned `postman/newman:alpine` container execution, Taskfile/Makefile runners with intelligent Docker fallback, deterministic CI exit codes, timeout constraints, and multi-reporter pipelines (CLI, JUnit XML, JSON, htmlextra). |

---

## Non-Negotiable Guardrails (Absolute Invariants)

1. **Mandatory Dedicated `README.md` (Living Runbook Mandate)**:
   - Every directory or project module housing Postman collections and Newman test suites (e.g., `newman/`, `postman/`, `tests/api/`, `e2e/api/`) MUST maintain a dedicated, up-to-date `README.md` directly in that root testing directory.
   - Any pull request or commit that introduces, alters, or removes collections, requests, environments, scripts, data fixtures, or CLI execution options MUST synchronize this `README.md` within the exact same atomic commit.
   - The `README.md` must document:
     - The target System Under Test (SUT) scope, microservice entrypoints, and API coverage.
     - Directory architecture tree detailing collections, environments, data fixtures, and reports.
     - Business scenario and test coverage matrix (positive happy-path, negative validation, error responses).
     - Environment selection matrix and variable contracts (`local`, `dev`, `staging`).
     - Execution runbook covering local CLI, task runner commands (Taskfile/Makefile), and containerized execution (Docker fallback).
     - Deterministic quality gates, latency SLAs, and assertion standards.
     - Synthetic data generation strategies and state teardown.
     - Multi-reporter outputs and CI/CD artifact paths.

2. **Absolute Ban on Hardcoded Secrets & Dynamic Environment Decoupling**:
   - Never commit sensitive API keys, bearer tokens, passwords, private client secrets, or production credentials inside `.postman_collection.json` or `.postman_environment.json` files.
   - Committed environment files MUST contain only non-sensitive baseline metadata (such as `baseUrl`) and empty/placeholder string values for credentials.
   - Secrets MUST be injected dynamically at runtime via Newman CLI parameters (`--env-var "apiKey=$SECRET_KEY"`), process environment variables, or CI secret store injection.
   - Every testing directory MUST provide an environment reference or `.env.example` defining all required runtime variables.

3. **Strict Modern `pm.*` API & Absolute Ban on Deprecated Legacy Syntax**:
   - Never use deprecated legacy Postman syntax (e.g., `tests["Status is 200"] = responseCode.code === 200`, `postman.setEnvironmentVariable(...)`, `postman.getEnvironmentVariable(...)`, `postman.clearEnvironmentVariable(...)`).
   - Use the modern `pm.*` API exclusively alongside Chai BDD assertions: `pm.test()`, `pm.response.to.have.status()`, `pm.response.to.be.json`, `pm.expect()`.
   - The sole permissible use of the legacy `postman` global object is execution workflow branching via `postman.setNextRequest(...)`.

4. **Defensive Response Parsing & Non-Skipping Assertions**:
   - Never call `pm.response.json()` blindly without validating that the response body is non-empty and has a JSON Content-Type (`pm.response.to.be.json`).
   - Never wrap entire assertion blocks inside conditional guards like `if (pm.response.code === 200) { ... }` that silently skip tests when an unexpected status occurs. Tests must fail explicitly and loudly when expectations are not met.
   - Use safe parsing helper patterns or try/catch blocks when extracting payload properties to prevent unhandled JavaScript exceptions from crashing collection execution.

5. **Formal JSON Schema Contract Validation**:
   - For all critical production endpoints, response structures MUST be validated against formal JSON Schemas using `pm.response.to.have.jsonSchema(schema)` (Ajv / tv4).
   - Validate field types, mandatory properties, and allowed enum values.
   - For error responses, explicitly assert RFC 7807 Problem Details schemas (`type`, `title`, `status`, `detail`).

6. **Dynamic Synthetic Data over Brittle Static Collisions**:
   - Never rely on static, hardcoded entity identifiers, emails, usernames, or phone numbers in state-mutating requests (`POST`, `PUT`, `PATCH`).
   - Leverage dynamic variables (`{{$guid}}`, `{{$randomEmail}}`, `{{$randomFirstName}}`, `{{$isoTimestamp}}`) or pre-request JavaScript logic to synthesize unique payloads per iteration.
   - Prevent flaky test collisions caused by unique constraint violations across repeated local or CI runs.

7. **Deterministic Workflow Branching & Fail-Safe Loop Guards**:
   - When using `postman.setNextRequest(requestName)` for dynamic branching, polling, or retry logic, always define an explicit terminal condition that concludes execution with `postman.setNextRequest(null)`.
   - Any retry or polling flow MUST implement an iteration counter guard (e.g., `maxRetries = 5`) to prevent infinite execution loops that freeze CI pipelines.

8. **Deterministic CI/CD Quality Gates & Exit Code Discipline**:
   - In automated CI/CD pipelines, Newman MUST never be executed with `--suppress-exit-code`. Any assertion failure, syntax error, or connection refusal must return a non-zero exit code to fail the pipeline gate.
   - Use `--bail` when fail-fast execution is mandated for critical smoke or regression checks.
   - Always declare explicit timeouts (`--timeout-request <ms>`, `--timeout-script <ms>`) to eliminate hanging pipelines caused by unreachable hosts.

9. **Multi-Reporter Discipline (Human-Readable & Machine-Readable)**:
   - For automated CI/CD execution, configure multi-reporters: standard CLI output (`--reporters cli`) for streaming logs, and machine-readable JUnit XML (`--reporters cli,junit --reporter-junit-export reports/newman-results.xml`) for native test reporting in CI platforms (GitHub Actions, GitLab CI, Jenkins).
   - Optional rich HTML dashboards (such as `htmlextra`) should be configured for artifact archiving.
   - Report destination directories (`reports/`) must be pre-created, gitignored, and kept clean across builds.

10. **Containerized Execution & Self-Contained Automation**:
    - API test suites must provide seamless containerized execution via official, pinned Docker images (`postman/newman:alpine`, never untracked `:latest`).
    - Provide task runner recipes (e.g., `Taskfile.yaml` or `Makefile`) featuring native CLI execution with automatic Docker fallback when the local `newman` binary is not found on the host machine.
    - Ensure cross-platform container networking: handle Linux host networking (`--network host`) vs. macOS/Windows host routing (`host.docker.internal` or Docker Compose service DNS).

Consult the corresponding skills for complete technical deep dives, configuration recipes, and reference implementations.
