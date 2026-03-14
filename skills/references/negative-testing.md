# Negative Testing Patterns

Error-path assertions, guard patterns, and conditional request flows for comprehensive Postman collections.

## Error-Path Assertions

Include test blocks for expected error responses alongside success cases.

### 401 — Unauthenticated

```javascript
pm.test('401 when missing auth token', () => {
    pm.response.to.have.status(401);
    const body = pm.response.json();
    pm.expect(body).to.have.property('message');
    pm.expect(body.message).to.include('Unauthenticated');
});
```

### 403 — Forbidden

```javascript
pm.test('403 when user lacks permission', () => {
    pm.response.to.have.status(403);
    const body = pm.response.json();
    pm.expect(body).to.have.property('message');
    pm.expect(body.message).to.include('Forbidden');
});
```

### 404 — Not Found

```javascript
pm.test('404 for nonexistent resource', () => {
    pm.response.to.have.status(404);
    const body = pm.response.json();
    pm.expect(body).to.have.property('message');
});
```

### 422 — Validation Error

```javascript
pm.test('422 on invalid input', () => {
    pm.response.to.have.status(422);
    const body = pm.response.json();
    pm.expect(body).to.have.property('message');
    pm.expect(body).to.have.property('errors').that.is.an('object');
    pm.expect(body.errors).to.have.property('email');
});
```

### 429 — Rate Limited

```javascript
pm.test('429 after exceeding rate limit', () => {
    pm.response.to.have.status(429);
    pm.expect(pm.response.headers.has('Retry-After')).to.be.true;
});
```

## Variable Chaining Guard Pattern

The silent chaining anti-pattern:

```javascript
// BAD: Silent failure — if creation fails, subsequent requests get undefined IDs
if (pm.response.code === 201) {
    const data = pm.response.json().data;
    if (data && data.id) {
        pm.collectionVariables.set('resource_id', data.id);
    }
}
```

**Problem**: When creation fails (e.g., 422 validation error), `if (data && data.id)` silently skips the set, leaving `resource_id` empty. Subsequent requests use empty values and fail cryptically.

**Fix**: Use `pm.test()` to surface broken chaining in Newman reports:

```javascript
pm.test('Resource created and ID captured', () => {
    pm.response.to.have.status(201);
    const body = pm.response.json();
    pm.expect(body).to.have.property('data').that.is.an('object');
    pm.expect(body.data).to.have.property('id');
    // Only set if test passes
    pm.collectionVariables.set('resource_id', body.data.id);
});
```

Now if creation fails, the test fails with a clear assertion error in the Newman report, rather than silently chaining a broken ID.

## Conditional Request Flows with `postman.setNextRequest()`

Control which request runs next based on response status or values.

### Stop Run on Setup Failure

```javascript
// In a setup request's test script
if (pm.response.code !== 201) {
    console.log('Setup failed, stopping run');
    postman.setNextRequest(null);
}
```

### Skip to Cleanup on Error

```javascript
// In a request's test script
if (pm.response.code >= 400) {
    console.log('Skipping dependent requests, jumping to cleanup');
    postman.setNextRequest('Cleanup - Delete Test Resource');
}
```

### Conditional Data Cleanup

```javascript
// In a read request
const shouldDelete = pm.response.json().data.status === 'draft';
if (shouldDelete) {
    postman.setNextRequest('Delete Resource');
} else {
    postman.setNextRequest('Publish Resource');
}
```

### Rules for `setNextRequest()`

- **Only works in Collection Runner and Newman** — Not in the Postman client manual run
- **`null` stops the entire run** — No more requests execute
- **Request names are case-sensitive** — Match exactly: `'Update User'` not `'Update user'`
- **Skipped requests don't run** — Their test scripts are not executed
- **Folder structure is ignored** — You can jump between folders by request name

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|---|---|---|
| Only happy-path tests | Silent failures in error cases | Include 401/403/404/422 tests |
| Silent variable chaining | Broken IDs not surfaced until later requests fail | Use `pm.test()` to assert capture succeeds |
| Using `setNextRequest` in pre-request scripts | Pre-request scripts run before the request sends, so the response isn't available yet | Use only in test scripts |
| Checking `if (data)` instead of asserting | Silently skips on malformed responses | Assert structure with `pm.expect()` first |
| Assuming Collection Runner behavior in Newman | `setNextRequest()` works differently in different contexts | Test in Newman before deploying to CI |
