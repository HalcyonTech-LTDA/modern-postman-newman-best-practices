---
description: Automatically enforces modern Postman and Newman automated API testing standards: collection craftsmanship and v2.1 modularity, modern pm.* Chai BDD assertions, defensive parsing and schema validation, zero-secret environment decoupling, dynamic request chaining and workflow control, containerized orchestration with official Docker runners, deterministic CI/CD quality gates, and mandatory living README.md runbook documentation discipline.
trigger: model_decision
---

# Modern Postman & Newman Engineering Guardrails & Skill Router

When authoring, reviewing, refactoring, or executing Postman collections, environments, pre-request/test scripts, Newman command-line executions, Docker container runners, CI/CD automated test pipelines, or API test documentation, strictly adhere to Modern Postman & Newman Engineering Standards and the corresponding specialized skills:

| When working on... | Consult & Activate Skill | Key Focus Areas |
| :--- | :--- | :--- |
| **Collection Design & Script Craftsmanship** | `modern-postman-collection-craftsmanship` | Collection v2.1 format, modular folder hierarchies, modern `pm.*` API, Chai BDD assertions, defensive parsing, JSON Schema validation, and workflow branching. |
| **Variable Scoping, Secrets & Fixtures** | `modern-postman-environments-and-secrets` | Five-tier variable scope hierarchy, zero hardcoded secrets, environment template contracts, runtime CLI secret injection, and data-driven testing. |
| **Living Documentation & Runbooks** | `modern-newman-living-documentation` | Mandatory dedicated `README.md` runbook, SUT scope, scenario & coverage matrices, environment contracts, and CLI/Docker execution recipes. |
| **Container Orchestration & CI/CD Gates** | `modern-newman-orchestration-and-ci` | Official pinned `postman/newman` container execution, Taskfile runners, deterministic exit codes, timeouts, and multi-reporter pipelines. |

---

## Non-Negotiable Guardrails (Absolute Invariants)

1. **Modern `pm.*` API & Absolute Legacy Ban**: Strictly prohibit deprecated legacy Postman syntax (e.g. `tests[...]`, `postman.setEnvironmentVariable`). Use the modern `pm.*` API exclusively alongside Chai BDD assertions (`pm.test()`, `pm.expect()`).
2. **Defensive Response Parsing & Loud Failures**: Always verify non-empty bodies and JSON headers before parsing response payloads. Never hide test assertions behind conditional status guards that silently pass on unexpected errors; tests must fail explicitly and loudly.
3. **Formal JSON Schema Contract Validation**: Response bodies of critical production endpoints must be validated against formal JSON Schemas (`jsonSchema`). Validate standard RFC 7807 Problem Details schemas for HTTP error responses.
4. **Dynamic Synthetic Data & Loop-Safe Workflow Chaining**: Use dynamic synthetic variables for state-mutating requests to eliminate collision flakiness. Workflows using `postman.setNextRequest` must define bounded iteration limits and terminate explicitly with `null`.
5. **Zero Committed Secrets & Environment Decoupling**: Never commit sensitive API keys, tokens, or credentials inside collection or environment JSON files. Inject credentials dynamically at runtime via CLI parameters or environment variables.
6. **Deterministic CI/CD Gates, Multi-Reporters & Living Runbooks**: Never suppress Newman failure exit codes in automated CI/CD pipelines (`--suppress-exit-code` is prohibited). Standardize execution with official pinned container runners, configure multi-reporters (CLI and JUnit XML), and synchronize a dedicated living `README.md` in every atomic test commit.

Consult the corresponding skills for complete technical deep dives, configuration recipes, and reference implementations.
