---
name: modern-postman-collection-craftsmanship
description: Use this skill when authoring, structuring, or refactoring Postman collections and scripts: Collection v2.1 modular architecture, folder hierarchy design, modern pm.* Chai BDD assertions, defensive response parsing, JSON Schema contract verification, dynamic synthetic data generation, and loop-safe workflow chaining (postman.setNextRequest).
---

# Modern Postman Collection Craftsmanship

This skill governs the code craftsmanship, script architecture, and assertion design of writing production-grade Postman collections for automated testing with Newman.

---

## 1. The Postman Request Execution Lifecycle

Every request inside a Postman collection executes through a structured, multi-phase lifecycle. Understanding how scripts execute within this flow is critical for building deterministic, isolated, and maintainable test suites:

```text
 ┌─────────────────────────────────────────────────────────────┐
 │ 1. Pre-Request Script (Collection Level)                    │
 │    - Global setup, correlation ID generation, timing metrics│
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 2. Pre-Request Script (Folder Level)                        │
 │    - Folder-specific auth token injection, state preparation│
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 3. Pre-Request Script (Request Level)                       │
 │    - Dynamic payload generation (GUIDs, timestamps, emails) │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 4. HTTP Request Execution                                   │
 │    - Newman dispatches network call to the target endpoint  │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 5. Test Script (Collection Level)                           │
 │    - Global assertions (SLA < 1000ms, standard headers)     │
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 6. Test Script (Folder Level)                               │
 │    - Domain assertions (JSON Content-Type, auth validations)│
 └──────────────────────────────┬──────────────────────────────┘
                                │
 ┌──────────────────────────────▼──────────────────────────────┐
 │ 7. Test Script (Request Level)                              │
 │    - Status codes, schema validation, entity data capture   │
 │    - Workflow branching via postman.setNextRequest(...)     │
 └─────────────────────────────────────────────────────────────┘
```

### Invariant Rules across the Lifecycle:
- **Inherit Don't Duplicate**: Place shared pre-request logic (such as setting an `X-Correlation-ID` header) at the collection or folder level, not copy-pasted into every individual request.
- **Isolate State Between Requests**: Use request-level or local variable scopes for ephemeral data; avoid leaking test artifacts across unrelated folders.
- **Fail Loudly**: Assertions must never be bypassed conditionally. If a dependency fails, downstream tests must report explicit failures.

---

## 2. Modular Collection & Folder Architecture

Avoid flat collections containing dozens of unorganized requests. Structure collections hierarchically by business domain, resource, or test lifecycle:

```text
collections/
└── ms-service-api.postman_collection.json
    ├── 01 - Health & Observability/     # Fast smoke checks (Actuator, ping, OpenAPI)
    │   ├── GET /actuator/health
    │   └── GET /v3/api-docs
    ├── 02 - Identity & Authentication/  # Token issuance, refresh, invalid credentials
    │   ├── POST /api/v1/auth/login (Success)
    │   └── POST /api/v1/auth/login (Invalid Credentials)
    ├── 03 - Customer Management/        # CRUD journeys, input validation, state transitions
    │   ├── POST /api/v1/customers (Create - Success)
    │   ├── GET /api/v1/customers/:id (Get By ID - Success)
    │   ├── GET /api/v1/customers/:id (Not Found - 404)
    │   └── POST /api/v1/customers (Blank Name - 400 Validation Error)
    └── 04 - Order Processing/           # Dependent cross-resource transactional flows
        ├── POST /api/v1/orders (Place Order)
        └── GET /api/v1/orders/:id (Verify Order Status)
```

### Request Naming Conventions:
Name requests explicitly with their HTTP verb, endpoint resource, and intended scenario behavior:
- ✅ `POST /customers - Success (201 Created)`
- ✅ `GET /customers/:id - Not Found (404)`
- ✅ `POST /orders - Validation Failure (400)`
- ❌ `Create Customer` (ambiguous HTTP verb and test intent)
- ❌ `Test 1` (opaque, zero diagnostic value in Newman CLI output)

---

## 3. Modern `pm.*` API vs. Deprecated Legacy Syntax

Postman deprecated the legacy `tests[...]` object and `postman.get/set*` functions in favor of the standardized `pm.*` API and Chai BDD assertion syntax. 

### The Deprecation Invariant:
Never author or review legacy Postman syntax. Always enforce modern `pm.*` patterns:

| Legacy Syntax (STRICTLY BANNED) | Modern `pm.*` Syntax (MANDATORY) |
| :--- | :--- |
| `tests["Status is 200"] = responseCode.code === 200;` | `pm.test("Status code is 200 OK", () => { pm.response.to.have.status(200); });` |
| `tests["Body contains error"] = responseBody.has("error");` | `pm.test("Body contains error", () => { pm.expect(pm.response.text()).to.include("error"); });` |
| `postman.setEnvironmentVariable("id", "123");` | `pm.environment.set("id", "123");` |
| `postman.getEnvironmentVariable("id");` | `pm.environment.get("id");` |
| `postman.clearEnvironmentVariable("id");` | `pm.environment.unset("id");` |
| `postman.setGlobalVariable("k", "v");` | `pm.globals.set("k", "v");` |

The **only** permissible use of the legacy `postman` namespace is workflow branching: `postman.setNextRequest(requestName)`.

---

## 4. Defensive Assertions & The Loud Failure Philosophy

### The "Silent Skipping" Antipattern
A common antipattern in Postman test suites is wrapping assertion blocks in conditional status checks:

```javascript
// ❌ CRITICAL ANTIPATTERN: Silent Skipping
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

if (pm.response && pm.response.code === 200) {
    pm.test("Health status is UP", function () {
        const jsonData = pm.response.json();
        pm.expect(jsonData.status).to.eql("UP");
    });
    pm.test("Response time is acceptable (< 300ms)", function () {
        pm.expect(pm.response.responseTime).to.be.below(300);
    });
}
```

**Why this is dangerous:**
If the endpoint returns `500 Internal Server Error`, only the first assertion executes and fails. The subsequent two tests (`Health status is UP` and `Response time is acceptable`) are **silently skipped** because `pm.response.code === 200` is false. The test report underreports failures and masks broken contractual expectations.

### The Modern Defensive Pattern
Every test must be registered with `pm.test()`. Inside each test, handle parsing safely so unexpected bodies fail the test with clear diagnostics rather than crashing or skipping:

```javascript
// ✅ PRODUCTION STANDARD: Unconditional Test Registration with Safe Parsing
pm.test("Status code is 200 OK", function () {
    pm.response.to.have.status(200);
});

pm.test("Response Content-Type is application/json", function () {
    pm.response.to.be.json;
});

pm.test("Health status is UP", function () {
    pm.response.to.have.status(200);
    pm.response.to.be.json;
    const jsonData = pm.response.json();
    pm.expect(jsonData).to.be.an("object");
    pm.expect(jsonData.status).to.eql("UP");
});

pm.test("Response latency SLA is met (< 300ms)", function () {
    pm.expect(pm.response.responseTime).to.be.below(300);
});
```

---

## 5. Formal JSON Schema Contract Validation

Verify API payloads against standard JSON Schemas to ensure backwards compatibility and strict adherence to API contracts:

```javascript
pm.test("Response conforms to Customer JSON Schema", function () {
    const customerSchema = {
        type: "object",
        required: ["id", "name", "email", "status", "createdAt"],
        properties: {
            id: { type: "string", format: "uuid" },
            name: { type: "string", minLength: 1, maxLength: 100 },
            email: { type: "string", format: "email" },
            status: { type: "string", enum: ["PENDING", "ACTIVE", "SUSPENDED"] },
            createdAt: { type: "string", format: "date-time" }
        },
        additionalProperties: true
    };

    pm.response.to.have.jsonSchema(customerSchema);
});
```

### RFC 7807 Problem Details Verification for Error Responses
For negative scenarios (HTTP 400, 404, 409, 422, 500), validate the standard RFC 7807 Problem Details structure:

```javascript
pm.test("Response conforms to RFC 7807 Problem Details specification", function () {
    pm.response.to.have.status(404);
    pm.response.to.be.json;

    const problemDetailSchema = {
        type: "object",
        required: ["type", "title", "status", "detail"],
        properties: {
            type: { type: "string" },
            title: { type: "string" },
            status: { type: "integer", minimum: 400, maximum: 599 },
            detail: { type: "string" },
            instance: { type: "string" }
        }
    };

    pm.response.to.have.jsonSchema(problemDetailSchema);
    
    const body = pm.response.json();
    pm.expect(body.status).to.eql(404);
    pm.expect(body.title).to.include("Not Found");
});
```

---

## 6. Dynamic Synthetic Data Generation

Hardcoded data values cause flaky collisions in automated test suites when unique database constraints are triggered (e.g. duplicate email addresses, duplicate IDs).

### Postman Dynamic Variables:
Use Postman's built-in dynamic variables directly in request templates:
- `{{$guid}}`: Universally unique identifier (UUID v4)
- `{{$randomEmail}}`: Realistic random email address
- `{{$randomFirstName}}` / `{{$randomLastName}}`: Random human names
- `{{$randomPhoneNumber}}`: Realistic phone number
- `{{$isoTimestamp}}`: Current ISO-8601 timestamp
- `{{$randomInt}}`: Random integer between 1 and 1000

### Pre-Request Script Data Generation:
When dynamic variables need to be reused across request headers, URL parameters, and request bodies within the same request lifecycle:

```javascript
// Pre-request script
const uniqueId = pm.variables.replaceIn("{{$guid}}");
const uniqueEmail = `test-user-${Date.now()}@example.com`;
const customerName = pm.variables.replaceIn("{{$randomFirstName}} {{$randomLastName}}");

pm.variables.set("reqCustomerId", uniqueId);
pm.variables.set("reqCustomerEmail", uniqueEmail);
pm.variables.set("reqCustomerName", customerName);
```

In the Request Body:
```json
{
  "id": "{{reqCustomerId}}",
  "name": "{{reqCustomerName}}",
  "email": "{{reqCustomerEmail}}"
}
```

---

## 7. Safe Workflow Branching & Loop Guards

Newman executes collection requests sequentially in the order they appear in the collection JSON. To implement non-linear flows (conditional routing, polling, retries), use `postman.setNextRequest(requestName)`.

### The Invariant: Mandatory Loop Guards & Terminal Null
Uncontrolled `setNextRequest` calls easily cause infinite loops that freeze Newman and exhaust CI worker minutes. Every dynamic workflow MUST implement:
1. An explicit terminal condition terminating with `postman.setNextRequest(null)`.
2. A strict iteration counter guard (e.g., `maxRetries = 5`).

```javascript
// Test script for a polling request: GET /orders/:id/status
const maxRetries = 5;
const currentAttempt = Number(pm.environment.get("orderPollAttempt") || 0) + 1;
pm.environment.set("orderPollAttempt", currentAttempt);

let res = {};
try {
    res = pm.response.json();
} catch (e) {
    // Non-JSON or empty response handled gracefully
}

if (res.status === "COMPLETED") {
    pm.test("Order reached COMPLETED status", () => {
        pm.expect(res.status).to.eql("COMPLETED");
    });
    // Reset attempt counter and continue normal flow
    pm.environment.unset("orderPollAttempt");
    postman.setNextRequest("04 - Next Business Step");
} else if (currentAttempt < maxRetries) {
    // Retry the same request
    postman.setNextRequest("Poll Order Status");
} else {
    // Max attempts exceeded: fail test and stop workflow
    pm.environment.unset("orderPollAttempt");
    pm.test("Order reached COMPLETED status within timeout", () => {
        pm.expect.fail(`Order remained in status '${res.status || "UNKNOWN"}' after ${maxRetries} polling attempts`);
    });
    postman.setNextRequest(null);
}
```

---

## Detailed Topic Guides

1. **Collection Architecture & Modular Design**:
   - Deep dive into collection v2.1 format, folder design patterns, and naming conventions.
   - [Read Guide](./references/collection_architecture_and_modular_design.md)

2. **Chai BDD Assertions & Schema Validation**:
   - Comprehensive assertion patterns, status codes, headers, SLA thresholds, and Ajv schemas.
   - [Read Guide](./references/pm_assertions_and_schema_validation.md)

3. **Scripting, Data Generation & Workflow Control**:
   - Pre-request lifecycle, Faker/dynamic variables, state passing, and fail-safe state machines.
   - [Read Guide](./references/scripting_data_generation_and_workflows.md)
