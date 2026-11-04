<div align="center">

# Modern Postman Newman Best Practices Plugin
### Automated API Contract Testing, Dynamic Collections & Headless CI/CD

[![Antigravity Plugin](https://img.shields.io/badge/Antigravity-Plugin%20v1.0.0-4F46E5.svg)](https://antigravity.google)
[![License: MIT](https://img.shields.io/badge/License-MIT-10B981.svg)](./LICENSE)
[![Organization](https://img.shields.io/badge/Organization-HalcyonTech--LTDA-black.svg)](https://github.com/HalcyonTech-LTDA)

<p align="center">
  <b>Professional modern Postman and Newman automated API testing best practices covering collection craftsmanship, dynamic workflows, zero-secret environment decoupling, deterministic CI/CD quality gates, containerized orchestration, and mandatory living README.md runbook documentation discipline.</b>
</p>

</div>

---

## Installation & Verification

### Antigravity CLI (`agy`)

Install directly into your Antigravity environment via the CLI:

```bash
agy plugin install https://github.com/HalcyonTech-LTDA/modern-postman-newman-best-practices
```

Verify that the plugin is active and adheres to the strict official schema:

```bash
agy plugin list
agy plugin validate --strict ./modern-postman-newman-best-practices
```

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

---

## What's Included (Component Catalog)

### Contextual Rules (`rules/`)
Automatically loaded to govern and protect project contexts:
- **[`modern-postman-newman-engineering.md`](rules/modern-postman-newman-engineering.md)**: Automatically enforces modern Postman and Newman automated API testing standards: collection craftsmanship and v2.1 modularity, modern pm.* Chai BDD assertions, defensive parsing and schema validation, zero-secret environment decoupling, dynamic request chaining and workflow control, containerized orchestration with official Docker runners, deterministic CI/CD quality gates, and mandatory living README.md runbook documentation discipline.

### Procedural Skills (`skills/`)
On-demand actionable workflows and runbooks:
- **[`/modern-newman-living-documentation`](skills/modern-newman-living-documentation/SKILL.md)**: Use this skill when authoring, updating, or reviewing documentation for Postman collections and Newman test suites: establishing the mandatory dedicated README.md living documentation, documenting System Under Test (SUT) scope, maintaining scenario & test coverage matrices, defining dynamic environment contracts, providing copy-pasteable execution runbooks (Taskfile, Make, CLI, Docker), and enforcing documentation synchronization in pull requests.
- **[`/modern-newman-orchestration-and-ci`](skills/modern-newman-orchestration-and-ci/SKILL.md)**: Use this skill when configuring, automating, or optimizing Newman test execution in containerized environments or CI/CD pipelines: official Docker runners (postman/newman:alpine), Taskfile/Makefile automation with Docker fallback, deterministic quality gates (exit codes, --bail), timeout controls, and multi-reporter configurations (CLI, JUnit XML, JSON, htmlextra).
- **[`/modern-postman-collection-craftsmanship`](skills/modern-postman-collection-craftsmanship/SKILL.md)**: Use this skill when authoring, structuring, or refactoring Postman collections and scripts: Collection v2.1 modular architecture, folder hierarchy design, modern pm.* Chai BDD assertions, defensive response parsing, JSON Schema contract verification, dynamic synthetic data generation, and loop-safe workflow chaining (postman.setNextRequest).
- **[`/modern-postman-environments-and-secrets`](skills/modern-postman-environments-and-secrets/SKILL.md)**: Use this skill when managing Postman variables, environment configuration files, sensitive secrets, or data-driven testing: five-tier variable scope hierarchy, zero-secret environment decoupling, runtime CLI injection (--env-var), iteration data fixtures (CSV/JSON), and variable sanitation.

---

## Directory Hierarchy

```text
modern-postman-newman-best-practices/
├── plugin.json              # Official Antigravity manifest (schema v1)
├── LICENSE                  # MIT License (Copyright 2026 HalcyonTech LTDA)
├── .gitignore               # System ignores & installed_version.json exclusions
├── .gitattributes           # Deterministic line endings (* text=auto eol=lf)
├── README.md                # Golden Standard documentation & runbook
├── assets/
│   └── logo.svg             # Official Halcyon Tech corporate vector symbol
├── rules/                   # Contextual rules discovered automatically
│   └── modern-postman-newman-engineering.md
├── skills/                  # Actionable procedural skills (Progressive Disclosure)
│   └── modern-newman-living-documentation/
│       └── SKILL.md
│   └── modern-newman-orchestration-and-ci/
│       └── SKILL.md
│   └── modern-postman-collection-craftsmanship/
│       └── SKILL.md
│   └── modern-postman-environments-and-secrets/
│       └── SKILL.md
```

---

## 💡 Prompt Recipes & Use Cases

```markdown
- "Design modular Postman collections with automated pre-request scripts and tests"
- "Configure headless Newman CI/CD test automation pipelines with reporting"
- "Audit API contract compliance and generate comprehensive environment configurations"
```

---

## 📄 License & Corporate Governance

Distributed under the **MIT License**. Copyright (c) 2026 **HalcyonTech LTDA**.  
Engineered with precision and excellence by [Halcyon Tech](https://github.com/HalcyonTech-LTDA).
