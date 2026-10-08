# Zero-Secret Injection & Environment Templates

This guide details best practices for decoupling secrets, creating secure environment templates, and injecting runtime credentials dynamically into Newman executions.

---

## 1. The Security Invariant

API test suites often require authentication credentials (API keys, client secrets, basic auth credentials, or signing tokens) to invoke secured endpoints.

### Invariant Rules:
1. **Never commit secrets to version control**: Never commit real passwords, production tokens, or private keys inside `.postman_collection.json` or `.postman_environment.json`.
2. **Commit sanitized templates**: Environment files committed to Git must only declare keys, non-sensitive endpoints (`baseUrl`), and empty/placeholder string values for credentials.
3. **Inject secrets at runtime**: Credentials must be injected dynamically into Newman at execution time.

---

## 2. Sanitized Environment Template Specification

Every environment JSON file should follow this secure template pattern:

```json
{
  "id": "b7c2a1d4-8e3f-412a-9f5c-2d8e4b1a7c3d",
  "name": "Staging",
  "values": [
    {
      "key": "baseUrl",
      "value": "https://staging-api.example.com",
      "type": "default",
      "enabled": true
    },
    {
      "key": "apiKey",
      "value": "",
      "type": "secret",
      "enabled": true
    },
    {
      "key": "clientSecret",
      "value": "",
      "type": "secret",
      "enabled": true
    }
  ],
  "_postman_variable_scope": "environment"
}
```

Notice that `apiKey` and `clientSecret` have `value: ""` and `type: "secret"`.

---

## 3. The Companion `.env.example` Pattern

To give engineers and CI systems a clear contract of what variables are required to run the test suite, maintain an `.env.example` in the test root:

```ini
# Newman API Test Suite Environment Contract
# Target base URL (overrides environment file if set)
API_BASE_URL=https://staging-api.example.com

# Authentication Credentials (REQUIRED for protected flows)
API_KEY=your_staging_api_key_here
CLIENT_SECRET=your_staging_client_secret_here

# Newman Reporting Options
NEWMAN_REPORTERS=cli,junit
NEWMAN_REPORT_DIR=reports
```

---

## 4. Injecting Credentials via Newman CLI

Use Newman's `--env-var` parameter to inject credentials into the active environment at runtime. You can specify `--env-var` multiple times:

```bash
newman run collections/api.postman_collection.json \
  -e environments/staging.postman_environment.json \
  --env-var "apiKey=${API_KEY}" \
  --env-var "clientSecret=${CLIENT_SECRET}"
```

### Dynamic Base URL Overrides:
To run the exact same collection against an ephemeral preview environment without modifying any JSON files:

```bash
newman run collections/api.postman_collection.json \
  -e environments/dev.postman_environment.json \
  --env-var "baseUrl=https://pr-421.preview.example.com"
```

---

## 5. Automated CI/CD Pipeline Secret Injection

### GitHub Actions Workflow Recipe:
```yaml
name: API Integration Tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  api-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Install Newman
        run: npm install -g newman

      - name: Execute API Tests via Newman
        env:
          STAGING_API_KEY: ${{ secrets.STAGING_API_KEY }}
          STAGING_CLIENT_SECRET: ${{ secrets.STAGING_CLIENT_SECRET }}
        run: |
          mkdir -p newman/reports
          newman run newman/collections/ms-service.postman_collection.json \
            -e newman/environments/staging.postman_environment.json \
            --env-var "apiKey=${STAGING_API_KEY}" \
            --env-var "clientSecret=${STAGING_CLIENT_SECRET}" \
            --reporters cli,junit \
            --reporter-junit-export newman/reports/results.xml
```

---

## 6. Dynamic Token Retrieval (OAuth2 / JWT Flows)

Rather than passing a long-lived static bearer token into Newman, implement a dedicated authentication request at the start of the collection:

1. Request: `01 - Auth / POST /oauth/token`
2. Request Body:
   ```json
   {
     "grant_type": "client_credentials",
     "client_id": "{{clientId}}",
     "client_secret": "{{clientSecret}}"
   }
   ```
3. Test Script:
   ```javascript
   pm.test("Token successfully acquired", () => {
       pm.response.to.have.status(200);
       const body = pm.response.json();
       pm.expect(body.access_token).to.be.a("string").that.is.not.empty;
       
       // Store token temporarily in environment for subsequent requests
       pm.environment.set("bearerToken", body.access_token);
   });
   ```
4. Subsequent Requests: Set Authorization header to `Bearer {{bearerToken}}`.
