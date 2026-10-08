---
name: modern-newman-living-documentation
description: Use this skill when authoring, updating, or reviewing documentation for Postman collections and Newman test suites: establishing the mandatory dedicated README.md living documentation, documenting System Under Test (SUT) scope, maintaining scenario & test coverage matrices, defining dynamic environment contracts, providing copy-pasteable execution runbooks (Taskfile, Make, CLI, Docker), and enforcing documentation synchronization in pull requests.
---

# Modern Newman Living Documentation Discipline

This skill defines the non-negotiable standard for maintaining living documentation for Postman and Newman automated API test suites. In modern cloud engineering and microservices architectures, an API test suite without a clear, living runbook decays into an unmaintainable, untrusted artifact that engineers avoid running or updating.

---

## 1. The Living Documentation Philosophy

### Why Undocumented API Test Suites Cause System Degradation:
1. **The "Black-Box" Runbook Antipattern**: API testing directories (e.g., `newman/`, `postman/`, `tests/api/`) frequently accumulate collections, environment files, and scripts with zero documentation on what environment variables they require (`API_BASE_URL`, `AUTH_TOKEN`), which microservices they target, or what mutations they execute.
2. **The Accidental Contamination Hazard**: Untrained engineers running a poorly documented Newman collection against a staging or pre-production environment can trigger destructive deletions, create hundreds of orphan database records, or spam external webhooks due to undocumented test data strategies.
3. **Silent Contract Drift**: As APIs evolve, unmaintained collections silently drift out of alignment with the actual services, continuing to test legacy deprecated routes while leaving newly released endpoints completely unverified.
4. **Developer Friction & Flaky Onboarding**: New team members or platform engineers cannot easily run suites locally because commands, prerequisite tokens, or Docker recipes are undocumented.

### The Invariant Rule: Dedicated `README.md` Mandate
Whenever a repository contains a Postman collection or Newman testing suite (e.g., `newman/`, `postman/`, `tests/api/`), it **MUST** maintain a dedicated, living `README.md` located directly in that root testing directory.

Any pull request or commit that introduces, alters, or removes a collection request, environment variable, data fixture, or CLI execution option **MUST** synchronize this `README.md` within the exact same atomic commit.

---

## 2. Anatomy of a Production-Grade Newman `README.md`

A professional Newman `README.md` consists of eight mandatory sections:

```text
newman/README.md
├── 1. Overview & Service Under Test (SUT) Scope
├── 2. Directory Architecture Tree
├── 3. Collection Modules & Endpoint Coverage Catalog (Scenario Matrix)
├── 4. Environment Selection & Configuration Matrix
├── 5. Execution Runbook (Taskfile / Make / Direct CLI / Docker Fallback)
├── 6. Quality Gates, SLAs & Assertion Standards
├── 7. Reporting Mechanisms (CLI, JSON, JUnit XML, HTML/htmlextra)
└── 8. Test Data Strategy & Teardown Discipline
```

### Detailed Breakdown of Sections:

### 1. Overview & Service Under Test (SUT) Scope
- Explicitly states which microservice, backend API, or gateway is under test.
- Links to relevant application source code (controllers, domain routers, OpenAPI definitions).
- Defines the purpose of the suite (e2e integration verification, regression quality gate, smoke test).

### 2. Directory Architecture Tree
- Provides an ASCII tree explaining the role of every directory and file:
  - `collections/`: Postman collection JSON definitions.
  - `environments/`: Sanitized environment definitions (local, dev, staging).
  - `fixtures/` (if present): External JSON/CSV data files for data-driven iterations.
  - `reports/`: Destination directory for output artifacts (`.gitkeep`).

### 3. Collection Modules & Endpoint Coverage Catalog (Scenario Matrix)
- Catalogs all logical modules and test scenarios in the collection:
  - Module / Folder Name (e.g., `01 - Actuator & Health`, `02 - Identity`, `03 - Customers`).
  - Target Endpoint & HTTP Verb (`POST /api/v1/customers`).
  - Scenario Type (Positive happy path, boundary validation, negative error handling).
  - Assertions & Key Contract Verifications (HTTP 201, `Location` header, JSON Schema validation, SLA < 300ms).

### 4. Environment Selection & Configuration Matrix
- A clear markdown table of all supported environments:
  - Environment Name (`local`, `dev`, `staging`).
  - Default Base URL (`http://localhost:8080`, `https://dev-api.example.com`).
  - Environment JSON file path (`newman/environments/local.postman_environment.json`).
  - Required runtime secrets (`API_KEY`, `CLIENT_SECRET`).

### 5. Execution Runbook
- Copy-pasteable, verified shell commands covering all execution modes:
  - **Task Runner (Taskfile / Makefile)**: Preferred native developer command (e.g., `task test:api ENV=staging`).
  - **Direct CLI Execution**: Local `newman run ...` command with reporter flags.
  - **Containerized Execution (Docker Fallback)**: Official `postman/newman:alpine` command with host networking and volume mounts for developers without Node.js.

### 6. Quality Gates, SLAs & Assertion Standards
- Documents latency budgets and SLA targets (e.g., P95 response time < 400ms).
- Documents fail-fast flags (`--bail`) and timeout policies (`--timeout-request`, `--timeout-script`).

### 7. Reporting Mechanisms
- How reports are generated and where they are placed:
  - CLI streaming reporter for real-time terminal output.
  - JUnit XML report (`reports/results.xml`) for automated CI pipeline test visualization.
  - JSON / HTML / htmlextra reports for detailed debugging and artifact archives.

### 8. Test Data Strategy & Teardown Discipline
- Explains synthetic data generation (`{{$guid}}`, `{{$randomEmail}}`).
- Documents state persistence (`pm.environment.set`) and cleanup/teardown mechanics.

---

## 3. Pull Request Synchronization Checklist

Before submitting or approving any pull request that touches API test collections or environments, verify:

- [ ] Does the dedicated `README.md` exist in the root testing directory?
- [ ] Were newly added endpoints, folders, or scenarios added to the Collection Modules & Coverage Catalog table?
- [ ] If a new environment variable or secret was introduced, is it documented in the Environment Selection Matrix and `.env.example`?
- [ ] Are all execution commands in the runbook up to date with current flags and filenames?
- [ ] Are report artifact paths documented and configured to point to a gitignored directory?
- [ ] Is the `README.md` update included in the exact same atomic commit as the collection/environment changes?

---

## Detailed Topic Guides

1. **Standard Newman `README.md` Template Blueprint**:
   - Production-ready, domain-agnostic Markdown template that can be copied directly into any project.
   - [Read Guide](./references/newman_readme_standard_template.md)

2. **Scenario Matrix & Runbook Recipes**:
   - Designing tabular scenario matrices, building Taskfile/Makefile runners with intelligent Docker fallback, and configuring cross-platform container networking.
   - [Read Guide](./references/scenario_matrix_and_runbook_recipes.md)
