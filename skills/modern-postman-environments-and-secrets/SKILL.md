---
name: modern-postman-environments-and-secrets
description: Use this skill when managing Postman variables, environment configuration files, sensitive secrets, or data-driven testing: five-tier variable scope hierarchy, zero-secret environment decoupling, runtime CLI injection (--env-var), iteration data fixtures (CSV/JSON), and variable sanitation.
---

# Modern Postman Environments, Secrets & Data-Driven Fixtures

This skill defines the standards for variable scope management, environment decoupling, zero-secret security, and data-driven testing with Newman.

---

## 1. The Five-Tier Variable Scope Hierarchy

Postman resolves variable references (`{{variableName}}`) by scanning five hierarchical scopes from narrowest (highest precedence) to broadest (lowest precedence):

```text
 ┌─────────────────────────────────────────────────────────────┐
 │ 1. Local Variables (pm.variables) - Highest Precedence      │
 │    - Ephemeral, exists strictly during current request      │
 └──────────────────────────────┬──────────────────────────────┘
                                │ overrides
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 2. Data Variables (pm.iterationData)                        │
 │    - Injected via external data fixtures (-d data.json/csv) │
 └──────────────────────────────┬──────────────────────────────┘
                                │ overrides
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 3. Environment Variables (pm.environment)                   │
 │    - Target-specific (local, dev, staging, prod)            │
 └──────────────────────────────┬──────────────────────────────┘
                                │ overrides
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 4. Collection Variables (pm.collectionVariables)            │
 │    - Shared across all requests within the same collection  │
 └──────────────────────────────┬──────────────────────────────┘
                                │ overrides
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 5. Global Variables (pm.globals) - Lowest Precedence        │
 │    - Shared across all collections in the workspace         │
 └─────────────────────────────────────────────────────────────┘
```

### Best Practice by Scope:
- **Local (`pm.variables.set`)**: Use for dynamic request payloads (generated GUIDs, calculated timestamps, one-off hashes). Automatically disposed when the request finishes.
- **Data (`pm.iterationData`)**: Use for parameterized testing tables (CSV/JSON rows) injected into Newman with `-d`.
- **Environment (`pm.environment.set`)**: Use for base URLs and cross-request state sharing during multi-step user journeys.
- **Collection (`pm.collectionVariables`)**: Use for collection-wide constants (default timeouts, common API version prefixes).
- **Global (`pm.globals`)**: Avoid in automated CI/CD suites to prevent hidden cross-collection dependencies.

---

## 2. The Absolute Zero-Secret Mandate

### The Vulnerability:
Committing `.postman_environment.json` files with actual API keys, Bearer tokens, passwords, or client secrets exposes production credentials to git logs, pull requests, and automated scanners.

### The Invariant Standard:
1. Committed environment files (`environments/*.postman_environment.json`) MUST contain ONLY non-sensitive properties (like `baseUrl`) and empty/placeholder string values for credentials.
2. Runtime secrets MUST be injected dynamically at execution time via Newman CLI parameters (`--env-var`), environment variables, or CI secret stores.
3. Every repository MUST provide a `.env.example` or documented environment template listing all expected keys.

### Example Sanitized Environment File:
```json
{
  "id": "e4f8d2b1-6a9c-4f1e-8b2a-3c5d7e9f1a2b",
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
      "key": "jwtToken",
      "value": "",
      "type": "secret",
      "enabled": true
    }
  ],
  "_postman_variable_scope": "environment"
}
```

---

## 3. Dynamic Secret Injection via Newman CLI

Never modify committed files to insert credentials before running tests. Use Newman's native `--env-var` parameter or environment overrides:

### Single Secret Injection:
```bash
newman run collections/api.postman_collection.json \
  -e environments/staging.postman_environment.json \
  --env-var "apiKey=${STAGING_API_KEY}" \
  --env-var "jwtToken=${STAGING_JWT_TOKEN}"
```

### Multiple Environment Variable Overrides:
```bash
newman run collections/api.postman_collection.json \
  -e environments/dev.postman_environment.json \
  --env-var "baseUrl=https://pr-142.preview.example.com" \
  --env-var "apiKey=${DEV_API_KEY}"
```

---

## 4. Data-Driven Testing with Iteration Data

To test multiple input boundary conditions (valid inputs, edge cases, illegal formats, SQL injection vectors) without duplicating requests:

### 1. External Fixture File (`fixtures/customer-validation.json`):
```json
[
  { "name": "", "email": "valid@example.com", "expectedStatus": 400, "errorField": "name" },
  { "name": "Valid Name", "email": "invalid-email-format", "expectedStatus": 400, "errorField": "email" },
  { "name": "John Doe", "email": "john.doe@example.com", "expectedStatus": 201, "errorField": null }
]
```

### 2. Request & Assertion Script:
In the Request Body:
```json
{
  "name": "{{name}}",
  "email": "{{email}}"
}
```

In the Test Script:
```javascript
const expectedStatus = pm.iterationData.get("expectedStatus");
const errorField = pm.iterationData.get("errorField");

pm.test(`Status code matches expected (${expectedStatus})`, function () {
    pm.response.to.have.status(expectedStatus);
});

pm.test("Response body matches expected contract", function () {
    pm.response.to.be.json;
    const body = pm.response.json();
    if (errorField) {
        pm.expect(body.detail || body.message || JSON.stringify(body)).to.include(errorField);
    } else {
        pm.expect(body).to.have.property("id");
    }
});
```

### 3. Newman Execution:
```bash
newman run collections/api.postman_collection.json \
  -e environments/local.postman_environment.json \
  -d fixtures/customer-validation.json
```

---

## 5. State Sanitation & Teardown Discipline

When user journeys write temporary state to `pm.environment` (such as `createdCustomerId`, `tempAuthToken`), downstream tests or subsequent runs might inadvertently pick up stale data.

### Cleanup Patterns:
1. **Unset Temporary Variables**: Clean up transient state in the final step of a flow or teardown request:
```javascript
// Teardown step in test script
pm.environment.unset("createdCustomerId");
pm.environment.unset("tempAuthToken");
```

2. **Use Ephemeral Scopes**: Whenever possible, use `pm.variables.set(...)` instead of `pm.environment.set(...)` if the variable only needs to live during the current request.

---

## Detailed Topic Guides

1. **Variable Scopes & Lifecycle**:
   - In-depth guide on Postman's 5 variable layers, priority rules, and memory lifecycle.
   - [Read Guide](./references/variable_scopes_and_lifecycle.md)

2. **Zero-Secret Injection & Templates**:
   - Environment templates, `.env.example` specifications, and CI/CD secret manager recipes.
   - [Read Guide](./references/zero_secret_injection_and_templates.md)

3. **Data-Driven Iterations & Fixtures**:
   - Designing JSON/CSV iteration datasets, boundary matrices, and Newman flags.
   - [Read Guide](./references/data_driven_iterations_and_fixtures.md)
