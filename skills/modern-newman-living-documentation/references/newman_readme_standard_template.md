# Standard Newman `README.md` Template Blueprint

Use this template as the standard blueprint when authoring or reviewing the dedicated `README.md` for a Postman / Newman test suite directory (e.g., `newman/README.md` or `tests/api/README.md`).

---

```markdown
# Automated API Testing with Newman

This directory contains the automated end-to-end API test suite for `<service-name>`, built with [Newman](https://github.com/postmanlabs/newman) (the Postman command-line collection runner).

---

## 1. Directory Structure

```text
newman/
├── README.md                                    # Living API testing guide and runbook
├── .env.example                                 # Dynamic environment variable contract
├── collections/
│   └── <service-name>.postman_collection.json   # Postman collection with tests & assertions
├── environments/
│   ├── local.postman_environment.json           # Target: http://localhost:8080
│   ├── dev.postman_environment.json             # Target: https://dev-api.example.com
│   └── staging.postman_environment.json         # Target: https://staging-api.example.com
├── fixtures/                                    # (Optional) Data-driven testing datasets (JSON/CSV)
│   └── sample-data.json
└── reports/
    └── .gitkeep                                 # Output destination for JUnit XML, JSON & HTML reports
```

---

## 2. Test Suite & Coverage Matrix

The API collection is organized into modular folders covering positive business journeys, edge cases, and error handling:

| Module / Folder | Method & Path | Scenario Description | Expected Status | Key Assertions |
| :--- | :--- | :--- | :--- | :--- |
| **01 - System & Health** | `GET /actuator/health` | Service liveness & readiness check | `200 OK` | `status == "UP"`, SLA < 200ms |
| **01 - System & Health** | `GET /v3/api-docs` | OpenAPI 3.x schema availability | `200 OK` | Valid OpenAPI schema structure |
| **02 - Resource Management** | `POST /api/v1/resources` | Create resource with synthetic data | `201 Created` | `Location` header, JSON Schema, ID captured |
| **02 - Resource Management** | `GET /api/v1/resources/:id` | Retrieve created resource by ID | `200 OK` | Data parity against creation payload |
| **02 - Resource Management** | `GET /api/v1/resources/:id` | Non-existent ID returns Problem Details | `404 Not Found` | RFC 7807 Problem Details schema |
| **02 - Resource Management** | `POST /api/v1/resources` | Blank required fields validation | `400 Bad Request` | RFC 7807 schema, validation errors array |
| **03 - Business Workflows** | `POST /api/v1/workflows` | Execute dependent transactional flow | `201 Created` | Transaction reference, status "COMPLETED" |

---

## 3. Environment Selection

Select the target environment using the `ENV` variable or corresponding environment file:

| Environment | Base URL | Environment File | Authentication / Secrets |
| :--- | :--- | :--- | :--- |
| **`local`** *(default)* | `http://localhost:8080` | `newman/environments/local.postman_environment.json` | None / local dev mock tokens |
| **`dev`** | `https://dev-api.example.com` | `newman/environments/dev.postman_environment.json` | `API_KEY`, `CLIENT_SECRET` via CLI |
| **`staging`** | `https://staging-api.example.com` | `newman/environments/staging.postman_environment.json` | `API_KEY`, `CLIENT_SECRET` via CLI |

---

## 4. Execution via Task Runner (Taskfile)

The root `Taskfile.yaml` (or `Makefile`) features automated native execution with an intelligent Docker fallback (`postman/newman:alpine`):

```bash
# Execute against local application (default: ENV=local)
task test:api

# Execute against specific remote environments
task test:api ENV=dev
task test:api ENV=staging

# Execute with fail-fast enabled
task test:api BAIL=true
```

---

## 5. Direct CLI Execution

If you have Newman installed locally (`npm install -g newman`):

```bash
# Basic run with CLI reporter against local environment
newman run newman/collections/<service-name>.postman_collection.json \
  -e newman/environments/local.postman_environment.json

# Run with JUnit XML and JSON report exports for CI
newman run newman/collections/<service-name>.postman_collection.json \
  -e newman/environments/local.postman_environment.json \
  --reporters cli,junit,json \
  --reporter-junit-export newman/reports/results.xml \
  --reporter-json-export newman/reports/results.json

# Inject runtime secrets dynamically
newman run newman/collections/<service-name>.postman_collection.json \
  -e newman/environments/staging.postman_environment.json \
  --env-var "apiKey=${STAGING_API_KEY}" \
  --env-var "clientSecret=${STAGING_CLIENT_SECRET}" \
  --reporters cli,junit \
  --reporter-junit-export newman/reports/results.xml
```

---

## 6. Standalone Docker Execution

Run Newman without installing Node.js or npm on your host machine using the official Docker image:

```bash
# Linux (Host networking)
docker run --rm -i \
  --network host \
  -v "$(pwd)/newman:/etc/newman" \
  postman/newman:alpine run "/etc/newman/collections/<service-name>.postman_collection.json" \
  -e "/etc/newman/environments/local.postman_environment.json" \
  --reporters cli

# macOS / Windows Docker Desktop (Bridge networking via host.docker.internal)
docker run --rm -i \
  -v "$(pwd)/newman:/etc/newman" \
  postman/newman:alpine run "/etc/newman/collections/<service-name>.postman_collection.json" \
  -e "/etc/newman/environments/local.postman_environment.json" \
  --env-var "baseUrl=http://host.docker.internal:8080" \
  --reporters cli
```

---

## 7. Quality Gates & SLA Standards

- **HTTP Status Codes**: Exact status assertions for both positive and negative flows.
- **Latency SLAs**: Individual requests assert latency budgets (`pm.expect(pm.response.responseTime).to.be.below(300)`).
- **Contract Enforcement**: Critical responses validate against JSON Schema schemas.
- **CI Build Gate**: Newman exits with a non-zero exit code if any assertion fails or an unexpected network error occurs.

---

## 8. Test Data Strategy & Teardown

- **Synthetic Data**: Pre-request scripts generate unique values using `{{$guid}}` and `{{$randomEmail}}` to prevent collision errors across test runs.
- **State Chaining**: Resource IDs created in upstream requests (`POST /resources`) are stored in temporary variables (`pm.environment.set("resourceId", res.id)`) and consumed in downstream verification requests.
- **Teardown**: The final request in the journey executes a `DELETE` request or calls `pm.environment.unset(...)` to clean up temporary state.
```
