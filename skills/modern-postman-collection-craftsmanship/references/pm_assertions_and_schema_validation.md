# Chai BDD Assertions & Schema Validation

This guide provides a comprehensive catalog of modern Chai BDD assertion patterns, defensive coding techniques, and JSON Schema contract verification for Postman and Newman test suites.

---

## 1. Chai BDD Assertion Mechanics in Postman

Postman bundles the Chai assertion library (`expect` and `should` styles). The modern standard exclusively mandates the `pm.expect()` BDD syntax inside `pm.test()` blocks.

### Core Anatomy:
```javascript
pm.test("Informative test name describing expected behavior", function () {
    // 1. Arrange & Act (implicit from response)
    // 2. Assert using Chai BDD
    pm.expect(actualValue).to.equal(expectedValue);
});
```

---

## 2. Status Code & Header Assertions

### Status Codes:
```javascript
// Exact status code
pm.test("Status is 200 OK", function () {
    pm.response.to.have.status(200);
});

// Semantic status helper
pm.test("Response is successful (2xx)", function () {
    pm.response.to.be.success;
});

// Multiple acceptable status codes (e.g., asynchronous processing or idempotent deletes)
pm.test("Status is either 200 OK or 204 No Content", function () {
    pm.expect(pm.response.code).to.be.oneOf([200, 204]);
});

// Client error verification
pm.test("Status is 404 Not Found", function () {
    pm.response.to.have.status(404);
});
```

### Headers:
```javascript
// Content-Type validation
pm.test("Content-Type is JSON", function () {
    pm.response.to.be.json;
    // or explicitly checking header string
    pm.expect(pm.response.headers.get("Content-Type")).to.include("application/json");
});

// Location header on 201 Created responses
pm.test("Location header points to newly created resource", function () {
    pm.response.to.have.status(201);
    pm.expect(pm.response.headers.has("Location")).to.be.true;
    pm.expect(pm.response.headers.get("Location")).to.match(/^\/api\/v1\/customers\/[a-f0-9-]+$/);
});
```

---

## 3. Response Latency & SLA Quality Gates

Automated API tests serve as latency benchmarks during continuous integration. Asserting SLAs detects performance regressions early:

```javascript
pm.test("Response latency SLA is met (< 400ms)", function () {
    pm.expect(pm.response.responseTime).to.be.below(400);
});

// Tagging slow endpoints with explicit diagnostic messages
pm.test("Endpoint response time conforms to P95 budget", function () {
    const maxBudgetMs = 600;
    pm.expect(
        pm.response.responseTime,
        `Expected response time of ${pm.response.responseTime}ms to be under ${maxBudgetMs}ms`
    ).to.be.below(maxBudgetMs);
});
```

---

## 4. Defensive Payload Parsing

When an unexpected error occurs (e.g. 502 Bad Gateway from Nginx or 500 HTML error page from a servlet container), calling `pm.response.json()` throws an unhandled JSON parsing exception that terminates test execution prematurely.

### Safe Extraction Pattern:
```javascript
pm.test("Response body is valid JSON with expected properties", function () {
    pm.response.to.be.json;
    
    let payload;
    try {
        payload = pm.response.json();
    } catch (e) {
        pm.expect.fail("Response body could not be parsed as JSON: " + e.message);
    }

    pm.expect(payload).to.be.an("object");
    pm.expect(payload).to.have.property("id");
    pm.expect(payload.id).to.be.a("string").and.not.be.empty;
});
```

---

## 5. JSON Schema Contract Verification

Validating responses against formal JSON Schemas prevents contract drift between producers and consumers:

### Positive Success Contract Example:
```javascript
pm.test("Customer response conforms to JSON Schema v4/v7", function () {
    const customerSchema = {
        "$schema": "http://json-schema.org/draft-07/schema#",
        "type": "object",
        "required": ["id", "name", "email", "status", "roles", "createdAt"],
        "properties": {
            "id": {
                "type": "string",
                "pattern": "^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$"
            },
            "name": {
                "type": "string",
                "minLength": 1,
                "maxLength": 100
            },
            "email": {
                "type": "string",
                "format": "email"
            },
            "status": {
                "type": "string",
                "enum": ["ACTIVE", "PENDING_VERIFICATION", "SUSPENDED"]
            },
            "roles": {
                "type": "array",
                "items": { "type": "string" },
                "minItems": 1
            },
            "createdAt": {
                "type": "string"
            }
        },
        "additionalProperties": false
    };

    pm.response.to.have.jsonSchema(customerSchema);
});
```

---

## 6. RFC 7807 Problem Details Verification

Modern RESTful microservices standardize error responses using RFC 7807 `application/problem+json`:

```javascript
pm.test("Error response conforms to RFC 7807 Problem Details specification", function () {
    pm.response.to.have.status(400);

    const problemDetailsSchema = {
        "type": "object",
        "required": ["type", "title", "status", "detail"],
        "properties": {
            "type": { "type": "string" },
            "title": { "type": "string" },
            "status": { "type": "integer", "enum": [400] },
            "detail": { "type": "string" },
            "instance": { "type": "string" },
            "invalidParams": {
                "type": "array",
                "items": {
                    "type": "object",
                    "required": ["name", "reason"],
                    "properties": {
                        "name": { "type": "string" },
                        "reason": { "type": "string" }
                    }
                }
            }
        }
    };

    pm.response.to.have.jsonSchema(problemDetailsSchema);

    const errorPayload = pm.response.json();
    pm.expect(errorPayload.status).to.eql(400);
    pm.expect(errorPayload.title).to.eql("Bad Request");
});
```

---

## 7. Deep Object & Array Equality Assertions

```javascript
// Array assertions
pm.test("Response contains expected list of items", function () {
    const data = pm.response.json();
    pm.expect(data.items).to.be.an("array").that.is.not.empty;
    pm.expect(data.totalElements).to.be.a("number").at.least(data.items.length);

    // Assert every item contains mandatory fields
    data.items.forEach((item, index) => {
        pm.expect(item, `Item at index ${index} must have an id`).to.have.property("id");
        pm.expect(item, `Item at index ${index} must be active`).to.have.property("status", "ACTIVE");
    });
});

// Deep equality against captured state
pm.test("Retrieved customer data matches registered values", function () {
    const actual = pm.response.json();
    const expectedName = pm.variables.get("customerName");
    const expectedEmail = pm.variables.get("customerEmail");

    pm.expect(actual.name).to.eql(expectedName);
    pm.expect(actual.email).to.eql(expectedEmail);
});
```
