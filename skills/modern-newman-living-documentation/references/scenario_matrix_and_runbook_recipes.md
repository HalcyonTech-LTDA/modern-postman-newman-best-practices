# Scenario Matrix & Runbook Recipes

This guide explains how to construct high-signal scenario matrices, craft cross-platform execution recipes, and automate Newman runs with intelligent Docker fallbacks.

---

## 1. Constructing the Scenario Matrix

The scenario matrix is the core catalog inside any Newman `README.md`. It translates opaque JSON collection definitions into a readable specification table for engineers, QA, and security auditors.

### Standard Schema:
```markdown
| Module / Folder | Method & Path | Scenario Description | Expected Status | Key Assertions & Notes |
| :--- | :--- | :--- | :--- | :--- |
| **01 - Auth** | `POST /auth/token` | Obtain OAuth2 Bearer token | `200 OK` | Validates JWT format & sets `bearerToken` |
| **01 - Auth** | `POST /auth/token` | Invalid client credentials | `401 Unauthorized` | RFC 7807 error model |
| **02 - Customers** | `POST /customers` | Register new customer | `201 Created` | Dynamic synthetic email, Location header, schema match |
| **02 - Customers** | `GET /customers/:id`| Retrieve created customer | `200 OK` | Verifies data parity against registration payload |
| **02 - Customers** | `GET /customers/:id`| Query non-existent ID | `404 Not Found` | Problem Details schema validation |
| **02 - Customers** | `POST /customers` | Blank name validation failure | `400 Bad Request` | Asserts field error array |
```

---

## 2. Taskfile Runner with Intelligent Docker Fallback

A best practice for polyglot engineering teams is encapsulating Newman test execution in a root `Taskfile.yaml`. If the developer has Newman installed locally, it runs at native speed; if not, it automatically falls back to Docker:

```yaml
version: '3'

vars:
  NEWMAN_DIR: 'newman'
  COLLECTION: 'newman/collections/api.postman_collection.json'
  DEFAULT_ENV: 'local'

tasks:
  test:api:
    desc: 'Run automated API integration tests via Newman (native CLI or Docker fallback)'
    vars:
      ENV_NAME: '{{.ENV | default .DEFAULT_ENV}}'
      ENVIRONMENT: 'newman/environments/{{.ENV_NAME}}.postman_environment.json'
      REPORT_XML: 'newman/reports/newman-results.xml'
      REPORT_JSON: 'newman/reports/newman-results.json'
      BAIL_FLAG: '{{if eq .BAIL "true"}}--bail{{else}}{{end}}'
    cmds:
      - mkdir -p newman/reports
      - |
        if command -v newman >/dev/null 2>&1; then
          echo "Running Newman natively via host CLI against environment: {{.ENV_NAME}}..."
          newman run "{{.COLLECTION}}" \
            -e "{{.ENVIRONMENT}}" \
            {{.BAIL_FLAG}} \
            --reporters cli,junit,json \
            --reporter-junit-export "{{.REPORT_XML}}" \
            --reporter-json-export "{{.REPORT_JSON}}"
        else
          echo "Newman not found on host. Falling back to Docker (postman/newman:alpine)..."
          
          # Detect OS for Docker host networking resolution
          DOCKER_NET=""
          BASE_URL_OVERRIDE=""
          if [ "{{.ENV_NAME}}" = "local" ]; then
            case "$(uname -s)" in
              Darwin*|MINGW*|MSYS*|CYGWIN*)
                # On macOS and Windows Docker Desktop, bridge networking with host.docker.internal
                DOCKER_NET=""
                BASE_URL_OVERRIDE="--env-var baseUrl=http://host.docker.internal:8080"
                ;;
              *)
                # On Linux, bind directly to host network
                DOCKER_NET="--network host"
                ;;
            esac
          fi
          
          docker run --rm -i \
            ${DOCKER_NET} \
            -v "$(pwd)/newman:/etc/newman" \
            postman/newman:alpine run "/etc/newman/collections/api.postman_collection.json" \
            -e "/etc/newman/environments/{{.ENV_NAME}}.postman_environment.json" \
            ${BASE_URL_OVERRIDE} \
            {{.BAIL_FLAG}} \
            --reporters cli,junit,json \
            --reporter-junit-export "/etc/newman/reports/newman-results.xml" \
            --reporter-json-export "/etc/newman/reports/newman-results.json"
        fi
```

---

## 3. Cross-Platform Container Networking Nuances

When executing Newman inside a Docker container against an application running on the host machine (`localhost:8080`):
- **Linux**: `--network host` allows the container to bind directly to the host network stack, resolving `http://localhost:8080` directly.
- **macOS & Windows (Docker Desktop)**: The Docker daemon runs inside a lightweight virtual machine. `--network host` does **not** map `localhost` to the Mac/Windows host. Instead, Docker Desktop provides a special internal DNS name: `http://host.docker.internal:8080`.

Always account for this in documentation and runner scripts by providing `--env-var "baseUrl=http://host.docker.internal:8080"` for Docker execution on macOS and Windows.

---

## 4. PR Synchronization Review Checklist

When reviewing pull requests that touch API test suites:
1. **Endpoint Parity**: If a new endpoint was added to the service, does a corresponding request exist in the collection?
2. **Scenario Matrix Sync**: Is the new request documented in the `README.md` scenario table with method, path, and expected status?
3. **Variable Sync**: Are all environment variables referenced in `{{...}}` declared in the `.env.example` file?
4. **Zero Secrets**: Are there any hardcoded keys, tokens, or credentials in the collection or environment JSON files?
5. **No Skipped Assertions**: Are all `pm.test` blocks unconditional, without silent-skipping `if (pm.response.code === 200)` wrappers?
6. **Report Cleanup**: Is the `reports/` output directory gitignored?
