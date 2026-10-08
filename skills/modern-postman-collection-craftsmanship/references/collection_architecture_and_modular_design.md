# Collection Architecture & Modular Design

This guide details best practices for structuring Postman collections for automated testing with Newman in modern microservices and API ecosystems.

---

## 1. Postman Collection v2.1.0 Standard

All modern Postman collections must conform to the Collection Format v2.1.0 schema (`https://schema.getpostman.com/json/collection/v2.1.0/collection.json`).

### Key Properties of Collection v2.1:
- **`info` block**: Defines the collection name, description, unique `_postman_id` (UUID), and schema URL.
- **`item` array**: Holds either folder objects (nested `item` arrays) or request objects.
- **`event` array**: Holds lifecycle script hooks (`prerequest` and `test`).
- **`variable` array**: Defines collection-scoped fallback variables.
- **`auth` block**: Defines collection-wide authentication protocols (Bearer Token, Basic Auth, API Key).

---

## 2. Organizational Patterns for API Collections

Collections should be organized following one of three established patterns, depending on the scope and maturity of the test suite:

### Pattern A: Resource/Domain-First (Recommended for Microservices)
Organize folders by aggregate root or bounded context:

```text
├── 01 - Health & Observability/
│   ├── GET /actuator/health
│   ├── GET /actuator/info
│   └── GET /v3/api-docs
├── 02 - Customers/
│   ├── POST /api/v1/customers (Create - Positive)
│   ├── GET /api/v1/customers/:id (Get By ID - Positive)
│   ├── PUT /api/v1/customers/:id (Update - Positive)
│   ├── DELETE /api/v1/customers/:id (Delete - Positive)
│   ├── GET /api/v1/customers/:id (Not Found - 404)
│   └── POST /api/v1/customers (Blank Email - 400)
└── 03 - Orders/
    ├── POST /api/v1/orders (Place Order)
    ├── GET /api/v1/orders/:id (Verify Placement)
    └── POST /api/v1/orders (Negative Amount - 422)
```

### Pattern B: Lifecycle & Test Tier (Recommended for E2E Suites)
Organize folders by test suite severity and scope:

```text
├── 01 - Smoke Tests/           # Critical path sanity tests (runs on every PR)
│   ├── Health Check
│   └── Public Status
├── 02 - Core Regression/       # Full functional CRUD and edge-case validations
│   ├── Customer Lifecycle
│   ├── Inventory Reservation
│   └── Payment Gateway
└── 03 - End-to-End Journeys/   # Complex cross-service flows mimicking user workflows
    └── Order-to-Cash Workflow
```

---

## 3. Request Naming Conventions & Taxonomy

In Newman execution reports (CLI, JUnit XML, HTML), request names are the primary human-readable identifiers. Unclear names waste engineering time during pipeline debugging.

### Standard Format:
`[METHOD] [PATH] - [SCENARIO_NAME] ([EXPECTED_STATUS])`

### Examples:
- ✅ `GET /actuator/health - Health check UP (200 OK)`
- ✅ `POST /customers - Register new customer (201 Created)`
- ✅ `GET /customers/{{customerId}} - Retrieve registered customer (200 OK)`
- ✅ `GET /customers/non-existent-id - Non-existent ID returns Problem Details (404 Not Found)`
- ✅ `POST /customers - Blank required fields validation error (400 Bad Request)`
- ❌ `Test Customer`
- ❌ `Customer 404`
- ❌ `New Request (Copy)`

---

## 4. Collection-Level and Folder-Level Script Inheritance

Do not repeat boilerplate configuration across hundreds of individual requests. Postman executes scripts hierarchically:

### Collection-Level Pre-Request Script:
Inject distributed tracing headers and timestamp markers globally:

```javascript
// Ensure correlation ID is present on every outgoing request
const correlationId = pm.variables.replaceIn("{{$guid}}");
pm.request.headers.upsert({
    key: "X-Correlation-ID",
    value: correlationId
});

// Record request dispatch timestamp for custom duration tracking
pm.variables.set("_reqStartTimestamp", Date.now());
```

### Collection-Level Test Script:
Enforce baseline technical standards across all endpoints in the collection:

```javascript
// Verify latency SLA across all requests in the collection
pm.test("Global SLA: Latency is below 1500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(1500);
});

// Ensure security headers are not missing
pm.test("Global Security: Response contains security headers", function () {
    pm.expect(pm.response.headers.has("X-Content-Type-Options")).to.be.true;
});
```

---

## 5. Parameterizing Paths and Query Parameters

Never concatenate URL path segments into raw strings. Use Postman URL path variables and query params:

### Good Practice:
- **Raw URL**: `{{baseUrl}}/api/v1/customers/:customerId`
- **Path Variable**: `customerId` -> `{{activeCustomerId}}`
- **Query Parameter**: `page` -> `1`, `size` -> `20`

This ensures Newman serializes URLs correctly across different operating systems and environments.
