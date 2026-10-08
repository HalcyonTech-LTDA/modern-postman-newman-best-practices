# Variable Scopes & Lifecycle in Postman & Newman

This guide details the behavior, lifecycle, precedence, and management of variable scopes in Postman and Newman.

---

## 1. Scope Precedence & Resolution Rules

When a variable like `{{baseUrl}}` or `{{customerId}}` is referenced in a URL, header, payload, or resolved programmatically via `pm.variables.get("key")`, Postman resolves it according to strict hierarchical precedence:

```text
Local > Data > Environment > Collection > Global
```

| Scope | Lifetime | Mutability in Script | Resolution Target |
| :--- | :--- | :--- | :--- |
| **Local** | Current request execution only | `pm.variables.set("k", "v")` | Single HTTP request lifecycle |
| **Data** | Current iteration row | Read-only (`pm.iterationData.get("k")`) | Newman `-d` fixture rows |
| **Environment** | Active environment session | `pm.environment.set("k", "v")` | Active environment file |
| **Collection** | Active collection | `pm.collectionVariables.set("k", "v")` | Current collection JSON |
| **Global** | Postman app workspace | `pm.globals.set("k", "v")` | Shared workspace |

---

## 2. When to Use Which Scope

### Local Scope (`pm.variables`)
- **Use for**: Ephemeral variables created in pre-request scripts, such as formatted timestamps, random UUIDs, or request signatures.
- **Why**: Automatically discarded as soon as the request and test scripts finish. Leaves zero pollution in environments.
- **Example**:
  ```javascript
  // Pre-request script
  pm.variables.set("requestTimestamp", Date.now());
  ```

### Data Scope (`pm.iterationData`)
- **Use for**: Input parameters and expected output values in data-driven test runs.
- **Why**: Immutable per iteration; automatically advances to the next data row on each iteration cycle.
- **Example**:
  ```javascript
  // Test script
  const expectedTaxRate = pm.iterationData.get("taxRate");
  ```

### Environment Scope (`pm.environment`)
- **Use for**: Target base URLs (`baseUrl`), port bindings, and state that MUST be passed from an upstream request to a downstream request (e.g. creating an entity and verifying it in the next request).
- **Why**: Bridges multiple requests across the collection execution.
- **Example**:
  ```javascript
  // Test script in POST /customers
  pm.environment.set("currentCustomerId", res.id);
  ```

### Collection Scope (`pm.collectionVariables`)
- **Use for**: Collection-wide constants (e.g., standard API version `/v1`, default timeout thresholds).
- **Why**: Portable with the collection JSON itself, avoiding the need for environment variables for static defaults.

---

## 3. Preventing Environment Pollution

A common antipattern is setting dozens of temporary variables in `pm.environment` and never removing them. Over time, or across repeated test runs in the Postman desktop app, environments become polluted with stale data.

### Sanitation Strategies:
1. **Explicit Unsetting in Teardowns**:
   ```javascript
   // In the last test of a flow or a teardown folder
   pm.environment.unset("currentCustomerId");
   pm.environment.unset("currentOrderId");
   ```
2. **Newman Ephemeral Run Isolation**:
   By default, Newman runs do **NOT** write modified environment variables back to the source JSON file on disk, unless the `--export-environment` flag is explicitly passed. This makes Newman inherently cleaner than GUI Postman runs. Keep this invariant intact.
