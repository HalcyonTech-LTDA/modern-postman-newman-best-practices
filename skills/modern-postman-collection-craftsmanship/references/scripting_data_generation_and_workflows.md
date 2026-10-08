# Scripting, Synthetic Data Generation & Workflows

This guide covers advanced scripting capabilities, dynamic synthetic data generation, and resilient workflow choreography using Postman and Newman.

---

## 1. The Postman JavaScript Sandbox

Postman executes scripts within a secure Node.js sandbox that provides standard JavaScript plus popular utility libraries pre-packaged:
- **`lodash` (`_`)**: Data manipulation and transformation utilities.
- **`crypto-js`**: Hashing and cryptography (HMAC, SHA-256, AES) for signature generation.
- **`moment`**: Date arithmetic and timestamp formatting.
- **`ajv` / `tv4`**: JSON schema validation engines.
- **`cheerio`**: HTML/XML DOM parsing.

---

## 2. Dynamic Synthetic Data Generation

Hardcoded payloads fail when databases enforce unique constraints (e.g., unique email addresses, order IDs, account numbers). Test data generation must be synthetic, unique, and dynamic.

### Dynamic Postman Variables:
Postman provides dozens of built-in `$random*` variables powered by Faker.js:
- `{{$guid}}`: UUID v4 string (e.g. `9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d`)
- `{{$randomEmail}}`: Realistic email address (e.g. `Jaylan_Waelchi73@gmail.com`)
- `{{$randomFirstName}}` / `{{$randomLastName}}`: Human names
- `{{$randomAlphaNumeric}}`: Random string
- `{{$randomPrice}}`: Random floating-point price
- `{{$timestamp}}`: Unix timestamp in seconds
- `{{$isoTimestamp}}`: ISO 8601 string

### Using `replaceIn()` in Pre-Request Scripts:
To generate and store dynamic variables before sending the request:

```javascript
// Pre-request script
const customerUuid = pm.variables.replaceIn("{{$guid}}");
const randomEmail = `perf-test-${Date.now()}-${pm.variables.replaceIn("{{$randomInt}}")}@example.com`;
const customerName = pm.variables.replaceIn("{{$randomFirstName}} {{$randomLastName}}");

// Store variables for the outgoing request
pm.variables.set("targetCustomerId", customerUuid);
pm.variables.set("targetCustomerEmail", randomEmail);
pm.variables.set("targetCustomerName", customerName);
```

Then reference them in the JSON body:
```json
{
  "id": "{{targetCustomerId}}",
  "name": "{{targetCustomerName}}",
  "email": "{{targetCustomerEmail}}"
}
```

---

## 3. Cryptographic Signature Generation in Pre-Request Scripts

For APIs requiring HMAC authentication, signature headers, or request timestamp validation:

```javascript
// Pre-request script: Generating HMAC-SHA256 signature
const secretKey = pm.environment.get("apiSecretKey") || "local-test-secret";
const timestamp = Math.floor(Date.now() / 1000).toString();
const payload = pm.request.body.raw ? pm.request.body.raw : "";
const message = `${pm.request.method}\n${pm.request.url.getPath()}\n${timestamp}\n${payload}`;

const signature = CryptoJS.HmacSHA256(message, secretKey).toString(CryptoJS.enc.Hex);

pm.request.headers.upsert({ key: "X-Timestamp", value: timestamp });
pm.request.headers.upsert({ key: "X-Signature", value: signature });
```

---

## 4. State Passing Across Requests

When tests execute multi-step journeys (e.g., Register Customer -> Get Customer -> Place Order -> Verify Status):

### Step 1: Capture in Test Script of Producer Request
```javascript
// POST /customers test script
pm.test("Status code is 201 Created", () => {
    pm.response.to.have.status(201);
});

const res = pm.response.json();
pm.expect(res).to.have.property("id");

// Persist customerId into environment for downstream consumer requests
pm.environment.set("createdCustomerId", res.id);
pm.environment.set("createdCustomerEmail", pm.variables.get("targetCustomerEmail"));
```

### Step 2: Consume in Subsequent Request
- Request URL: `{{baseUrl}}/api/v1/customers/{{createdCustomerId}}`
- In consumer test script:
```javascript
// GET /customers/:id test script
pm.test("Retrieved customer ID matches created ID", () => {
    const res = pm.response.json();
    const expectedId = pm.environment.get("createdCustomerId");
    pm.expect(res.id).to.eql(expectedId);
});
```

---

## 5. Safe Workflow Control with `postman.setNextRequest()`

By default, Newman executes collection requests in top-to-bottom order. Using `postman.setNextRequest(requestName)` alters this execution sequence.

### Invariant Rules for Workflow Control:
1. **Loop Guards**: Always enforce a maximum iteration count (`maxAttempts`) when implementing polling or retry flows.
2. **Deterministic Termination**: The terminal step must call `postman.setNextRequest(null)` if execution should stop early, or explicitly route to the next named request.
3. **Target by Name**: Always specify the exact request name as declared in the collection.

### Pattern: Polling with Exponential Backoff Guard

```javascript
// Test Script for GET /jobs/:id/status
const MAX_POLL_ATTEMPTS = 6;
const pollAttempt = Number(pm.environment.get("_jobPollAttempt") || 0) + 1;
pm.environment.set("_jobPollAttempt", pollAttempt);

pm.test("Status code is 200 OK", () => {
    pm.response.to.have.status(200);
});

let body = {};
try {
    body = pm.response.json();
} catch (e) {
    // Non-JSON or empty response handled gracefully
}

if (body.status === "COMPLETED") {
    pm.test("Job successfully completed", () => {
        pm.expect(body.status).to.eql("COMPLETED");
        pm.expect(body.resultUrl).to.be.a("string").that.is.not.empty;
    });

    // Cleanup state and continue
    pm.environment.unset("_jobPollAttempt");
    postman.setNextRequest("Download Result File");

} else if (body.status === "FAILED") {
    // Immediate failure
    pm.environment.unset("_jobPollAttempt");
    pm.test("Job failed with error", () => {
        pm.expect.fail(`Job processing failed: ${body.errorMessage || "Unknown error"}`);
    });
    postman.setNextRequest(null);

} else if (pollAttempt < MAX_POLL_ATTEMPTS) {
    // Repeat the current polling request
    postman.setNextRequest("Poll Job Status");

} else {
    // Max attempts exhausted: fail test and abort collection run
    pm.environment.unset("_jobPollAttempt");
    pm.test("Job completed within polling timeout", () => {
        pm.expect.fail(`Job timed out in status '${body.status || "UNKNOWN"}' after ${MAX_POLL_ATTEMPTS} attempts`);
    });
    postman.setNextRequest(null);
}
```
