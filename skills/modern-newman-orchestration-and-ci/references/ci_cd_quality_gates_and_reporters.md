# CI/CD Quality Gates & Reporters

This guide details Newman exit code semantics, timeout controls, reporter configurations, and CI/CD quality gate recipes.

---

## 1. Newman Exit Code Semantics

Newman is designed to integrate natively into automated CI/CD pipelines through POSIX exit codes:
- **`0` (Success)**: All requests completed, all Chai assertions passed, and no unhandled script exceptions were thrown.
- **`1` (Failure)**: At least one assertion failed, a syntax error occurred in a script, or an unhandled exception was thrown.
- **`Non-zero` (Network / System Error)**: DNS resolution failure, connection timeout, or invalid CLI flag.

### The Invariant: Ban on `--suppress-exit-code`
Passing `--suppress-exit-code` causes Newman to exit with code `0` even if every single assertion in the collection failed. In automated pipelines, this masks critical regressions and allows broken builds to pass into production. It is strictly prohibited in CI/CD.

---

## 2. Fail-Fast Mechanics with `--bail`

For large regression suites or high-velocity pull request validation, waiting for hundreds of downstream requests after a critical failure is wasteful.

### Granular Bail Options:
- `--bail`: Stops collection execution immediately on the first test failure or error.
- `--bail folder`: Stops execution of the current folder when a test fails, continuing to subsequent folders.
- `--bail failure`: Continues running requests, but skips subsequent tests within the failing request.

In pull request validation gates, prefer `--bail` to fail fast and provide instant feedback to developers.

---

## 3. Strict Timeout Budgets

Unresponsive APIs, deadlocked databases, or connection drops can cause Newman to hang indefinitely if timeouts are not configured.

### Recommended Timeout Configuration:
```bash
newman run collections/api.postman_collection.json \
  -e environments/ci.postman_environment.json \
  --timeout-request 5000 \
  --timeout-script 2000 \
  --timeout 120000
```

- `--timeout-request 5000`: Fails any request exceeding 5,000 milliseconds (5 seconds).
- `--timeout-script 2000`: Fails any pre-request or test script exceeding 2,000 milliseconds.
- `--timeout 120000`: Hard stop after 2 minutes for the entire collection run.

---

## 4. Multi-Reporter Configuration

Combine machine-readable and human-readable reporters:

```bash
newman run collections/api.postman_collection.json \
  -e environments/ci.postman_environment.json \
  --reporters cli,junit,json \
  --reporter-junit-export reports/newman-results.xml \
  --reporter-json-export reports/newman-results.json
```

### Reporter Overview:

| Reporter | Flag / Option | Destination | Purpose |
| :--- | :--- | :--- | :--- |
| **`cli`** | `--reporters cli` | STDOUT | Live streaming progress in CI logs. |
| **`junit`** | `--reporter-junit-export <path>` | XML file | Parsed by GitHub Actions / GitLab CI for native test result tabs. |
| **`json`** | `--reporter-json-export <path>` | JSON file | Machine-readable metrics for telemetry and dashboards. |
| **`htmlextra`** | `--reporter-htmlextra-export <path>` | HTML file | Rich interactive visual report for stakeholders. |

### CLI Noise Reduction in CI:
For high-volume suites where verbose CLI tables clutter logs:
```bash
# Display only failures in the CLI output while preserving full JUnit reports
newman run collections/api.postman_collection.json \
  -e environments/ci.postman_environment.json \
  --reporters cli,junit \
  --reporter-cli-no-success-assertions \
  --reporter-junit-export reports/junit.xml
```

---

## 5. CI/CD Pipeline Recipes

### GitLab CI (`.gitlab-ci.yml`):
```yaml
stages:
  - test

newman_api_tests:
  stage: test
  image:
    name: postman/newman:alpine
    entrypoint: [""]
  variables:
    NEWMAN_REPORT_DIR: "newman/reports"
  script:
    - mkdir -p ${NEWMAN_REPORT_DIR}
    - newman run newman/collections/service.postman_collection.json \
        -e newman/environments/staging.postman_environment.json \
        --env-var "apiKey=${CI_API_KEY}" \
        --timeout-request 5000 \
        --reporters cli,junit \
        --reporter-junit-export "${NEWMAN_REPORT_DIR}/junit.xml"
  artifacts:
    when: always
    reports:
      junit: newman/reports/junit.xml
    paths:
      - newman/reports/
```

### GitHub Actions Workflow:
```yaml
name: API Integration Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm install -g newman
      - run: |
          mkdir -p newman/reports
          newman run newman/collections/api.postman_collection.json \
            -e newman/environments/local.postman_environment.json \
            --reporters cli,junit \
            --reporter-junit-export newman/reports/junit.xml
      - uses: mikepenz/action-junit-report@v4
        if: always()
        with:
          report_paths: 'newman/reports/junit.xml'
```
