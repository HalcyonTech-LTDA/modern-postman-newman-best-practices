---
name: modern-newman-orchestration-and-ci
description: Use this skill when configuring, automating, or optimizing Newman test execution in containerized environments or CI/CD pipelines: official Docker runners (postman/newman:alpine), Taskfile/Makefile automation with Docker fallback, deterministic quality gates (exit codes, --bail), timeout controls, and multi-reporter configurations (CLI, JUnit XML, JSON, htmlextra).
---

# Modern Newman Orchestration & CI/CD Pipelines

This skill defines the standards for containerized execution, automated task runners, deterministic CI/CD quality gates, and multi-reporter configurations for Newman.

---

## 1. Official Containerized Execution (`postman/newman:alpine`)

Running Newman inside a lightweight container guarantees deterministic execution across development workstations and CI runners without requiring local Node.js or npm installations.

### Official Image Standards:
- **Base Image**: Always pin the official Alpine-based image: `postman/newman:alpine`.
- **Never Use Floating Unpinned Tags**: Avoid unpinned major versions or mutable `:latest` in production pipelines.
- **Volume Mounting**: Mount the project test directory into `/etc/newman` inside the container:
  ```bash
  docker run --rm -i \
    -v "$(pwd)/newman:/etc/newman" \
    postman/newman:alpine run "/etc/newman/collections/api.postman_collection.json" \
    -e "/etc/newman/environments/local.postman_environment.json" \
    --reporters cli
  ```

---

## 2. Cross-Platform Container Networking

When running tests from inside the Docker container against an application listening on the host machine:
- **Linux Hosts**: Use `--network host`. The container accesses services directly via `http://localhost:8080`.
- **macOS & Windows Docker Desktop**: `--network host` does not route traffic to the host's localhost. Instead, use Docker Desktop's internal host bridge: `http://host.docker.internal:8080`.
- **Docker Compose Stacks**: Join the shared user-defined network and address services by service name (e.g. `http://api-service:8080`).

---

## 3. Deterministic CI/CD Quality Gates & Exit Codes

In automated build pipelines, Newman serves as a hard quality gate. A broken assertion must halt the pipeline immediately.

### Non-Negotiable Invariants:
1. **Absolute Ban on `--suppress-exit-code`**: Never pass `--suppress-exit-code` in CI/CD. Newman must return exit code `1` (or non-zero) if any assertion fails or an uncaught script exception occurs.
2. **Fail-Fast with `--bail`**: Use `--bail` for pull request smoke checks or critical integration gates so Newman stops immediately upon the first assertion failure, conserving CI build minutes.
3. **Mandatory Timeout Budgets**: Always specify explicit timeouts to prevent hung jobs when a service is unresponsive or experiencing a deadlock:
   - `--timeout-request <ms>`: Maximum duration allowed for individual HTTP requests (e.g., 5000ms).
   - `--timeout-script <ms>`: Maximum duration allowed for pre-request/test script execution (e.g., 2000ms).
   - `--timeout <ms>`: Maximum total execution duration for the entire collection run.

```bash
newman run collections/api.postman_collection.json \
  -e environments/ci.postman_environment.json \
  --bail \
  --timeout-request 5000 \
  --timeout-script 2000
```

---

## 4. Multi-Reporter Discipline for CI Pipelines

Different consumers of test results require different reporting formats. Production pipelines should always configure multi-reporters:

```bash
newman run collections/api.postman_collection.json \
  -e environments/ci.postman_environment.json \
  --reporters cli,junit,json \
  --reporter-junit-export reports/newman-results.xml \
  --reporter-json-export reports/newman-results.json
```

### Purpose of Each Reporter:
- **`cli`**: Streams real-time human-readable progress directly into the terminal or CI build log.
- **`junit`**: Emits standard JUnit XML (`reports/newman-results.xml`), allowing CI platforms (GitHub Actions, GitLab CI, Azure DevOps) to render native test summary widgets, highlight failing tests, and track flaky trends over time.
- **`json`**: Produces machine-readable execution data for custom metrics dashboards or test telemetry ingestion.
- **`htmlextra`** (Optional): Generates a rich, interactive single-page HTML report for visual stakeholder sign-offs:
  ```bash
  newman run collections/api.postman_collection.json \
    -e environments/staging.postman_environment.json \
    --reporters cli,htmlextra \
    --reporter-htmlextra-export reports/newman-dashboard.html
  ```

---

## 5. Automated Pipeline Integration (GitHub Actions)

```yaml
name: Newman API Quality Gate

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test-api:
    name: Run Newman API Tests
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install Newman & Reporters
        run: npm install -g newman newman-reporter-htmlextra

      - name: Run Newman Test Suite
        env:
          API_KEY: ${{ secrets.CI_API_KEY }}
        run: |
          mkdir -p newman/reports
          newman run newman/collections/api.postman_collection.json \
            -e newman/environments/ci.postman_environment.json \
            --env-var "apiKey=${API_KEY}" \
            --timeout-request 5000 \
            --timeout-script 2000 \
            --reporters cli,junit,htmlextra \
            --reporter-junit-export newman/reports/junit-results.xml \
            --reporter-htmlextra-export newman/reports/test-dashboard.html

      - name: Publish JUnit Test Results
        uses: mikepenz/action-junit-report@v4
        if: always()
        with:
          report_paths: 'newman/reports/junit-results.xml'

      - name: Archive HTML Test Report
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: newman-html-report
          path: newman/reports/test-dashboard.html
```

---

## Detailed Topic Guides

1. **Container Orchestration & Docker Runners**:
   - Deep dive into official `postman/newman:alpine` container configuration, volume mapping, user permissions, and host networking.
   - [Read Guide](./references/container_orchestration_and_docker_runners.md)

2. **CI/CD Quality Gates & Reporters**:
   - Exit code handling, fail-fast mechanics, timeout strategies, and multi-reporter setups for CI/CD.
   - [Read Guide](./references/ci_cd_quality_gates_and_reporters.md)
