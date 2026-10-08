# Data-Driven Iterations & Fixtures

This guide explains how to design, organize, and execute data-driven test suites using Newman iteration data fixtures (JSON and CSV).

---

## 1. Why Data-Driven API Testing?

When testing input validation, edge cases, boundary values, or combinatorial inputs, creating separate Postman requests for each permutation results in collection bloat and massive maintenance overhead.

Data-driven testing decouples the test logic (request configuration and assertions) from test parameters (inputs and expected outputs). Newman runs the collection once for each row in the data fixture.

---

## 2. Fixture Formats: JSON vs. CSV

### Format A: JSON Array (Recommended for Structured Payloads)
Provides native typing (numbers, booleans, nested objects, nulls):

```json
[
  {
    "scenario": "Valid Standard Customer",
    "payload": {
      "name": "Alice Smith",
      "email": "alice@example.com",
      "age": 28
    },
    "expectedStatus": 201,
    "expectError": false
  },
  {
    "scenario": "Underage Customer - Boundary Failure",
    "payload": {
      "name": "Minor User",
      "email": "minor@example.com",
      "age": 17
    },
    "expectedStatus": 422,
    "expectError": true,
    "errorDetail": "Customer must be at least 18 years old"
  },
  {
    "scenario": "Malformed Email Format",
    "payload": {
      "name": "Bob Jones",
      "email": "invalid-email-no-domain",
      "age": 35
    },
    "expectedStatus": 400,
    "expectError": true,
    "errorDetail": "Invalid email address format"
  }
]
```

### Format B: CSV (Convenient for Tabular Test Matrices)
Simple key-value rows, but all values are parsed initially as strings:

```csv
scenario,name,email,age,expectedStatus,expectError
Valid Standard Customer,Alice Smith,alice@example.com,28,201,false
Underage Customer,Minor User,minor@example.com,17,422,true
Malformed Email,Bob Jones,invalid-email,35,400,true
```

---

## 3. Implementing the Test in Postman

### Request Body Configuration:
If using a structured JSON fixture, serialize the nested object in a pre-request script:

```javascript
// Pre-request script
const rowPayload = pm.iterationData.get("payload");
if (rowPayload) {
    pm.variables.set("requestBody", JSON.stringify(rowPayload));
}
```

In the Request Body (Raw JSON):
```text
{{requestBody}}
```

### Test Assertions Script:
```javascript
const scenario = pm.iterationData.get("scenario");
const expectedStatus = Number(pm.iterationData.get("expectedStatus"));
const expectError = Boolean(pm.iterationData.get("expectError"));
const errorDetail = pm.iterationData.get("errorDetail");

pm.test(`[${scenario}] Status code matches expected: ${expectedStatus}`, function () {
    pm.response.to.have.status(expectedStatus);
});

pm.test(`[${scenario}] Response payload satisfies contract requirements`, function () {
    pm.response.to.be.json;
    const body = pm.response.json();
    
    if (expectError) {
        if (errorDetail) {
            pm.expect(body.detail || body.message || JSON.stringify(body)).to.include(errorDetail);
        } else {
            pm.expect(body).to.satisfy(b => Boolean(b.error || b.detail || b.title));
        }
    } else {
        pm.expect(body).to.have.property("id");
    }
});
```

---

## 4. Running with Newman

Use the `-d` (or `--iteration-data`) flag:

```bash
newman run collections/customer-validation.postman_collection.json \
  -e environments/local.postman_environment.json \
  -d fixtures/customer-boundaries.json
```

### Limiting Iterations during Debugging:
Use `-n` (or `--iteration-count`) to run only the first $N$ rows:

```bash
# Run only the first 2 rows
newman run collections/customer-validation.postman_collection.json \
  -e environments/local.postman_environment.json \
  -d fixtures/customer-boundaries.json \
  -n 2
```
