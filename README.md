# Modern Postman & Newman Best Practices Plugin

An Antigravity plugin providing automated engineering guardrails, architectural standards, scripting craftsmanship, container orchestration, and procedural skills for modern **Postman & Newman** automated API testing: modular Collection v2.1 craftsmanship, modern `pm.*` Chai BDD assertions, defensive response parsing, JSON Schema contract verification, five-tier variable scope discipline, zero-secret environment decoupling, deterministic CI/CD quality gates, official container orchestration, and the **mandatory living `README.md` documentation discipline**.

---

## Installation

### Antigravity

Install directly in Antigravity via the CLI:

```bash
agy plugin install https://github.com/HalcyonTech-LTDA/modern-postman-newman-best-practices
```

Verify that the plugin is installed and ready:

```bash
agy plugin list
```

### Claude Code

Add the Halcyon Tech marketplace once, then install the plugin:

```text
/plugin marketplace add HalcyonTech-LTDA/halcyon-plugins
/plugin install modern-postman-newman-best-practices@halcyon-plugins
```

Skills are namespaced by plugin in Claude Code (`/modern-postman-newman-best-practices:<skill-name>`). The guardrails in `rules/` load through the skills in `claude/skills/`.

---
## Overview

In modern cloud-native systems, continuous delivery, and distributed microservices architectures, automated API testing is a critical quality gate. When microservices communicate over REST/HTTP, rapid and deterministic automated integration suites ensure services adhere to contractual obligations before reaching production.

However, Postman and Newman implementations frequently suffer from severe engineering antipatterns:
- **Monolithic, Unmaintainable Collections**: Gigantic collection JSON files with hundreds of flat, unstructured requests, duplicate scripts, and brittle sequencing.
- **Silent Assertion Skipping**: Wrapping assertions in conditional status checks (`if (pm.response.code === 200) { ... }`), causing broken endpoints returning 500 Internal Server Error to silently skip inner assertions rather than failing loudly.
- **Hardcoded Secrets & Environment Leaks**: Storing API keys, JWT tokens, database passwords, or private client secrets inside source-controlled `.postman_environment.json` files, resulting in severe security breaches.
- **Brittle Data Collisions**: Hardcoding static emails, IDs, and usernames in mutation endpoints (`POST /customers`), leading to test flakiness and unique constraint collisions across test runs.
- **Uncontrolled Polling Loops**: Using `postman.setNextRequest()` without deterministic termination guards or retry counters, locking CI workers into infinite execution loops until job timeouts occur.
- **The "Black-Box" Testing Antipattern**: Unmaintained test directories lacking documentation, leaving teams oblivious to which endpoints are covered, what environment variables are required, or how to execute suites locally and in containers.
- **Flaky Docker & CI Pipelines**: Using `--suppress-exit-code` in CI, relying on untracked `:latest` container images, or misconfiguring cross-platform container networking (`--network host` failing on macOS/Windows Docker Desktop).

This plugin establishes modern, production-grade engineering standards for Postman and Newman across four foundational pillars:

1. **Collection Craftsmanship & Script Discipline**: Structuring collections with modular folder hierarchies, writing expressive Chai BDD assertions via the modern `pm.*` API, validating JSON Schemas and RFC 7807 Problem Details, generating dynamic synthetic data, and choreographing safe workflow branching.
2. **Variable Scoping, Secrets & Data-Driven Fixtures**: Enforcing Postman's five-tier variable hierarchy, keeping committed environment files strictly secret-free, injecting runtime credentials via CLI arguments or CI secret stores, and running parameterized data-driven suites with iteration fixtures.
3. **Mandatory Living Documentation**: Requiring every API testing directory to maintain an up-to-date, comprehensive `README.md` acting as a living runbook with System Under Test (SUT) scope, scenario & coverage matrices, dynamic environment contracts, execution recipes (CLI, Taskfile, Docker), and PR synchronization checklists.
4. **Container Orchestration & CI/CD Quality Gates**: Standardizing official pinned `postman/newman:alpine` container execution with intelligent Docker fallbacks, asserting request and script timeouts, establishing deterministic exit codes, and generating multi-reporter outputs (streaming CLI, JUnit XML for CI gates, rich HTML dashboards).

---

## What's Included

### 1. Automatic Guardrails ([`rules/modern-postman-newman-engineering.md`](./rules/modern-postman-newman-engineering.md))
Automatically enforced across all projects utilizing Postman and Newman:
- **Mandatory Dedicated `README.md` (Living Runbook Mandate)**: Absolute mandate requiring any directory housing Postman collections to maintain an up-to-date living runbook synchronized in every PR.
- **Absolute Ban on Hardcoded Secrets**: Strictly forbidding committed secrets in collection or environment JSON files; enforcing dynamic runtime injection.
- **Strict Modern `pm.*` API**: Absolute ban on deprecated legacy syntax (`tests[...]`, `postman.setEnvironmentVariable(...)`) in favor of Chai BDD assertions.
- **Defensive Response Parsing & Non-Skipping Assertions**: Banning silent assertion skipping in `if` checks; requiring safe JSON parsing and loud failure propagation.
- **Formal JSON Schema Contract Validation**: Requiring JSON Schema (Ajv/tv4) validation for critical endpoints and RFC 7807 problem details for errors.
- **Dynamic Synthetic Data over Brittle Collisions**: Requiring dynamic variables (`{{$guid}}`, `{{$randomEmail}}`) or programmatic generation in mutating requests.
- **Deterministic Workflow Branching & Loop Guards**: Enforcing explicit exit conditions (`postman.setNextRequest(null)`) and retry counter guards.
- **Deterministic CI/CD Quality Gates**: Banning `--suppress-exit-code` in pipelines; enforcing non-zero exit codes on test failures and explicit request/script timeouts.
- **Multi-Reporter Discipline**: Requiring machine-readable JUnit XML alongside human-readable CLI/HTML reporters for CI integration.
- **Containerized Execution & Self-Contained Automation**: Providing pinned `postman/newman:alpine` container recipes with Taskfile/Makefile runners and cross-platform networking.

---

### 2. Specialized Skills (`skills/`)

#### 🛠️ [`modern-postman-collection-craftsmanship`](./skills/modern-postman-collection-craftsmanship/SKILL.md)
Guides collection architecture, modern Chai BDD assertions, defensive scripting, schema validation, and dynamic workflows:
- [`references/collection_architecture_and_modular_design.md`](./skills/modern-postman-collection-craftsmanship/references/collection_architecture_and_modular_design.md): Collection v2.1 structure, folder modularity (feature, resource, or lifecycle), naming conventions, and request chaining.
- [`references/pm_assertions_and_schema_validation.md`](./skills/modern-postman-collection-craftsmanship/references/pm_assertions_and_schema_validation.md): Modern Chai BDD assertion library, status code checks, header and SLA validation, JSON Schema validation (Ajv/tv4), and RFC 7807 Problem Details validation.
- [`references/scripting_data_generation_and_workflows.md`](./skills/modern-postman-collection-craftsmanship/references/scripting_data_generation_and_workflows.md): Pre-request vs. test script lifecycles, dynamic variables (`{{$guid}}`, `{{$randomEmail}}`, `{{$randomInt}}`), state passing between requests, and loop-safe workflow branching with `postman.setNextRequest`.

#### 🔐 [`modern-postman-environments-and-secrets`](./skills/modern-postman-environments-and-secrets/SKILL.md)
Guides variable scope management, zero-secret environment decoupling, runtime credential injection, and data-driven testing:
- [`references/variable_scopes_and_lifecycle.md`](./skills/modern-postman-environments-and-secrets/references/variable_scopes_and_lifecycle.md): The five variable scopes (Global, Collection, Environment, Data, Local), precedence rules, transient vs. persistent variables, and state contamination prevention.
- [`references/zero_secret_injection_and_templates.md`](./skills/modern-postman-environments-and-secrets/references/zero_secret_injection_and_templates.md): Sanitized `.postman_environment.json` templates, `.env.example` contracts, runtime CLI parameter injection (`--env-var`), and CI/CD secret manager integration.
- [`references/data_driven_iterations_and_fixtures.md`](./skills/modern-postman-environments-and-secrets/references/data_driven_iterations_and_fixtures.md): Data-driven testing with `-d / --iteration-data` (JSON/CSV), row-by-row iteration assertions, and boundary test fixtures.

#### 📖 [`modern-newman-living-documentation`](./skills/modern-newman-living-documentation/SKILL.md)
Guides the mandatory living documentation standard for Postman and Newman testing suites:
- [`references/newman_readme_standard_template.md`](./skills/modern-newman-living-documentation/references/newman_readme_standard_template.md): Complete, production-grade template blueprint for a dedicated `newman/README.md`.
- [`references/scenario_matrix_and_runbook_recipes.md`](./skills/modern-newman-living-documentation/references/scenario_matrix_and_runbook_recipes.md): Tabular scenario and endpoint coverage matrix, copy-pasteable CLI execution recipes, Taskfile/Makefile runners with intelligent Docker fallback, cross-platform container networking, and PR synchronization checklists.

#### 🐳 [`modern-newman-orchestration-and-ci`](./skills/modern-newman-orchestration-and-ci/SKILL.md)
Guides containerization, task runner automation, CI/CD pipeline integration, and multi-reporter configurations:
- [`references/container_orchestration_and_docker_runners.md`](./skills/modern-newman-orchestration-and-ci/references/container_orchestration_and_docker_runners.md): Official `postman/newman:alpine` Docker images, volume mounting, cross-platform host networking (`host.docker.internal` vs `--network host`), and file permissions.
- [`references/ci_cd_quality_gates_and_reporters.md`](./skills/modern-newman-orchestration-and-ci/references/ci_cd_quality_gates_and_reporters.md): Newman exit code semantics, timeout flags, multi-reporter configuration (`cli`, `junit`, `json`, `htmlextra`), and GitHub Actions/GitLab CI pipeline definitions.

---

## Directory Layout

```text
modern-postman-newman-best-practices/
├── plugin.json
├── README.md
├── rules/
│   └── modern-postman-newman-engineering.md
└── skills/
    ├── modern-postman-collection-craftsmanship/
    │   ├── SKILL.md
    │   └── references/
    │       ├── collection_architecture_and_modular_design.md
    │       ├── pm_assertions_and_schema_validation.md
    │       └── scripting_data_generation_and_workflows.md
    ├── modern-postman-environments-and-secrets/
    │   ├── SKILL.md
    │   └── references/
    │       ├── variable_scopes_and_lifecycle.md
    │       ├── zero_secret_injection_and_templates.md
    │       └── data_driven_iterations_and_fixtures.md
    ├── modern-newman-living-documentation/
    │   ├── SKILL.md
    │   └── references/
    │       ├── newman_readme_standard_template.md
    │       └── scenario_matrix_and_runbook_recipes.md
    └── modern-newman-orchestration-and-ci/
        ├── SKILL.md
        └── references/
            ├── container_orchestration_and_docker_runners.md
            └── ci_cd_quality_gates_and_reporters.md
```

---

## When to Use This Plugin

This plugin should be consulted whenever you are:
1. Creating or modifying Postman collections (`.postman_collection.json`) or environments (`.postman_environment.json`).
2. Writing or refactoring pre-request scripts or test assertion scripts.
3. Establishing automated API test suites executed by Newman in local development or CI/CD pipelines.
4. Setting up containerized test runners using Docker and task runners (Taskfile, Makefile).
5. Reviewing pull requests that introduce, modify, or delete API test endpoints, environments, or execution runbooks.
6. Authoring or synchronizing the mandatory living `README.md` documentation within an API testing directory.

---

## License

This project is licensed under the [MIT License](LICENSE) - see the LICENSE file for details.
Copyright (c) 2026 Halcyon Tech.
