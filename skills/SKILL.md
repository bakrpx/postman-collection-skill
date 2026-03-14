---
name: postman-collection
description: "Manages Postman API collections for Laravel projects including collection creation, test assertions, variable chaining, response examples, dynamic data, auth flows, Newman CI/CD, and Postman cloud sync. Use when creating or structuring Postman collections, writing test assertions, setting up variable chaining between requests, generating response examples, writing pre-request scripts, configuring Newman runs, or syncing with Postman cloud. Triggers on 'postman collection', 'newman run', 'postman tests', 'collection variables', 'response examples', 'postman auth', 'postman cloud sync', 'pre-request script', or 'postman ci'."
tags: [laravel, php, postman, api-testing, newman]
---

# Postman Collection Management for Laravel

Generic patterns for creating, testing, and deploying Postman collections against Laravel APIs.

## When to Apply

Activate this skill when:

- Creating or structuring a Postman collection for a Laravel API
- Writing test assertions or variable chaining between requests
- Adding response examples (success, error, validation)
- Writing pre-request scripts for auth or dynamic data
- Configuring Newman for CI/CD pipeline testing
- Syncing a local collection with Postman cloud
- Managing auth flows (JWT, Sanctum, API keys) in collections
- Generating dynamic test data to prevent uniqueness violations

## Directory Convention

```
postman/
├── collections/
│   └── <ProjectName>.postman_collection.json
├── environments/
│   ├── Local.postman_environment.json
│   └── Dev.postman_environment.json
└── globals/
    └── workspace.postman_globals.json
```

- One collection JSON per project
- One environment file per deployment target
- Globals for cross-environment constants

## Collection Creation Workflow

When building a collection for a Laravel API:

1. **Discover routes** — Run `php artisan route:list --json` to get all registered endpoints
2. **Map to folders** — Group routes by prefix into collection folders (e.g., `Admin API`, `Public API`)
3. **Set defaults** — Add `Accept: application/json` and `Content-Type: application/json` headers at collection level. **Without `Accept: application/json`, Laravel returns HTML error pages instead of JSON — this breaks all test assertions.**
4. **Add `base_url` variable** — Define `{{base_url}}` as a collection variable, set the actual value in environment files
5. **Build requests** — For each route, create a request with the correct method, URL pattern, headers, and sample body
6. **Add auth** — Configure folder-level auth inheritance (see Auth Patterns section)
7. **Add tests** — Write test assertions for each request (see Test Assertion Patterns)
8. **Add examples** — Generate response examples for documentation (see Response Examples)
9. **Verify** — Run with Newman: `npx newman run <collection> -e <env> --delay-request 100`

## Collection Structure

A Postman v2.1 collection JSON has this anatomy:

```json
{
  "info": { "name": "...", "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json" },
  "item": [
    {
      "name": "Folder Name",
      "item": [
        {
          "name": "Request Name",
          "request": { "method": "GET", "url": {}, "header": [], "body": {} },
          "response": [],
          "event": [
            { "listen": "prerequest", "script": { "type": "text/javascript", "exec": [] } },
            { "listen": "test", "script": { "type": "text/javascript", "exec": [] } }
          ]
        }
      ]
    }
  ],
  "auth": { "type": "noauth" },
  "variable": [{ "key": "base_url", "value": "", "type": "string" }]
}
```

Key rules:
- **`event.script.exec`** is an array of strings, one per line of JavaScript — not a single string
- **Auth inheritance** flows Collection -> Folder -> Request. Set `"type": "noauth"` at collection level, configure real auth per folder
- **Variable resolution order**: Local (`pm.variables`) > Collection (`pm.collectionVariables`) > Environment (`pm.environment`) > Global (`pm.globals`)
- **`{{variable_name}}`** syntax works in URLs, headers, and body content

### Script Execution Order

1. Collection pre-request -> 2. Folder pre-request -> 3. Request pre-request -> **Request sent** -> 4. Request test -> 5. Folder test -> 6. Collection test

## Test Assertion Patterns

Standard test blocks for every request:

```javascript
// Status code
pm.test('Status code is 200', () => {
    pm.response.to.have.status(200);
});

// Response time
pm.test('Response time < 2000ms', () => {
    pm.expect(pm.response.responseTime).to.be.below(2000);
});

// Content-Type (skip for 204 No Content)
pm.test('Content-Type is JSON', () => {
    pm.response.to.have.header('Content-Type', /application\/json/);
});
```

### Pagination Validation (List Endpoints)

Detect list endpoints by name pattern: `/^(List|Search|Index) /i.test(name) && method === 'GET'`.

```javascript
pm.test('Has valid pagination', () => {
    const json = pm.response.json();
    pm.expect(json).to.have.property('data').that.is.an('array');
    pm.expect(json).to.have.property('meta');
    pm.expect(json.meta).to.have.property('current_page');
    pm.expect(json.meta).to.have.property('per_page');
    pm.expect(json.meta).to.have.property('total');
});
```

### Single Resource Validation

```javascript
pm.test('Has valid resource', () => {
    const json = pm.response.json();
    pm.expect(json).to.have.property('data').that.is.an('object');
    pm.expect(json.data).to.have.property('id');
});
```

### Variable Chaining

Capture IDs from create responses for use in subsequent requests:

```javascript
// In test script of a Create endpoint
if (pm.response.code === 201) {
    const data = pm.response.json().data;
    if (data && data.id) {
        pm.collectionVariables.set('user_id', data.id);
    }
}
```

Ensure collection-level variables exist for each chained value so Newman can resolve them.

For error-path assertions, conditional request flows, and guard patterns to surface broken chaining, see `references/negative-testing.md`.

### Variable Scope Rules

| Scope | Set With | Lifetime | Use For |
|-------|----------|----------|---------|
| Local | `pm.variables.set()` | Current request only | Temp computed values |
| Collection | `pm.collectionVariables.set()` | Persists across requests | IDs, tokens |
| Environment | `pm.environment.set()` | Environment file | base_url, secrets |
| Global | `pm.globals.set()` | Globals file | Cross-environment constants |

## Response Examples

### Naming Convention

Use `"<code> - <status>"` format: `"200 - OK"`, `"201 - Created"`, `"422 - Validation Error"`.

### Response Object Structure

```json
{
  "name": "200 - OK",
  "originalRequest": { "method": "GET", "url": {}, "header": [] },
  "status": "OK",
  "code": 200,
  "_postman_previewlanguage": "json",
  "header": [{ "key": "Content-Type", "value": "application/json" }],
  "cookie": [],
  "body": "{ \"data\": { \"id\": 1, \"name\": \"Example\" } }"
}
```

For 204 responses: empty `body`, empty `header`, set `_postman_previewlanguage` to `"text"`.

### Success Responses

| Action | Code | Body Structure |
|--------|------|---------------|
| Show/List (GET) | 200 | `{ "data": {...} }` or `{ "data": [...], "meta": {...}, "links": {...} }` |
| Create (POST) | 201 | `{ "data": {...} }` |
| Update (PUT/PATCH) | 200 | `{ "data": {...} }` |
| Delete (DELETE) | 204 | Empty |

### Laravel Paginated Response

```json
{
  "data": [{ "id": 1, "name": "..." }],
  "links": { "first": "...?page=1", "last": "...?page=5", "prev": null, "next": "...?page=2" },
  "meta": { "current_page": 1, "from": 1, "last_page": 5, "path": "...", "per_page": 15, "to": 15, "total": 73 }
}
```

### Error Responses (Include for Authenticated Endpoints)

| Code | Name | Body |
|------|------|------|
| 401 | `401 - Unauthenticated` | `{ "message": "Unauthenticated." }` |
| 403 | `403 - Forbidden` | `{ "message": "Forbidden." }` |
| 404 | `404 - Not Found` | `{ "message": "Not found." }` |
| 429 | `429 - Too Many Requests` | `{ "message": "Too many requests." }` |

### 422 Validation Error Format

```json
{
  "message": "The name field is required. (and 2 more errors)",
  "errors": {
    "name": ["The name field is required."],
    "email": ["The email field is required."]
  }
}
```

Include 422 examples for any endpoint with a request body. Build the `errors` object from the endpoint's required fields.

## Dynamic Data

Prevent uniqueness violations on repeated Newman runs by generating unique values in pre-request scripts:

```javascript
const ts = Date.now();
const rand = Math.random().toString(36).substring(2, 6);
pm.variables.set('_email', `test-${ts}-${rand}@example.com`);
pm.variables.set('_slug', `item-${ts}-${rand}`);
pm.variables.set('_code', `CODE${rand.toUpperCase()}`);
```

Reference in request body: `"email": "{{_email}}"`.

Use `pm.variables.set()` (local scope) for dynamic values — they don't persist and won't pollute collection state.

| Value Type | Pattern |
|------------|---------|
| Email | `` `test-${ts}-${rand}@example.com` `` |
| Slug | `` `item-${ts}-${rand}` `` |
| Code/SKU | `` `CODE${rand.toUpperCase()}` `` |
| Name | `` `Test Item ${ts}` `` |

## Auth Patterns

Multi-layer Laravel APIs often need separate auth per layer. Key patterns:

- **Token caching**: Check expiry before re-fetching — avoids unnecessary auth requests
- **`noauth` inheritance**: Set collection-level auth to `noauth`, configure real auth per folder to prevent token leakage between layers
- **Multi-role token storage**: Use separate variables per layer (`admin_access_token`, `tenant_access_token`, `user_token`, `api_key`)
- **Secrets in environments**: Store credentials in environment files, never in collection variables

For detailed JWT acquisition, token refresh, Sanctum flows, and API key patterns, see `references/auth-patterns.md`.

## Variable Chaining & Environment Management

For parameterized URL resolution, environment file management, and data-driven testing with Newman, see `references/variable-chaining.md`.

## Newman CI/CD

### Basic Run

```bash
npx newman run postman/collections/MyAPI.postman_collection.json \
    -e postman/environments/Local.postman_environment.json \
    --delay-request 100
```

### Essential Flags

| Flag | Purpose |
|------|---------|
| `-e <file>` | Environment file |
| `-g <file>` | Globals file |
| `--delay-request <ms>` | Delay between requests (prevents rate limiting) |
| `--timeout-request <ms>` | Per-request timeout |
| `--bail` | Stop on first failure |
| `-r cli,json,htmlextra` | Multiple output reporters (requires `npm i -g newman-reporter-htmlextra`) |
| `--reporter-json-export <file>` | Save JSON report |
| `--folder "<name>"` | Run specific folder only |
| `--env-var "key=value"` | Override environment variable |

### Exit Codes

- `0` — All tests passed
- `1` — Test assertion failed or collection could not be loaded

### GitHub Actions Workflow

```yaml
name: API Tests
on: [push, pull_request]

jobs:
  newman:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm install -g newman newman-reporter-htmlextra
      - name: Start application
        run: |
          docker compose up -d
          sleep 10
      - name: Run Newman tests
        run: |
          newman run postman/collections/MyAPI.postman_collection.json \
              -e postman/environments/CI.postman_environment.json \
              --delay-request 100 --bail \
              -r cli,json,htmlextra --reporter-json-export newman-results.json
      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: newman-results
          path: |
            newman-results.json
            newman/
```

## Postman Cloud Sync

### Upload Collection

```bash
# Get collection UID
COLLECTION_UID=$(curl -s https://api.getpostman.com/collections \
    -H "X-Api-Key: $POSTMAN_API_KEY" \
    | jq -r '.collections[] | select(.name=="My API") | .uid')

# PUT replaces the entire collection
curl -X PUT "https://api.getpostman.com/collections/$COLLECTION_UID" \
    -H "X-Api-Key: $POSTMAN_API_KEY" \
    -H "Content-Type: application/json" \
    -d "{\"collection\": $(cat collection.json)}"
```

### ID Preservation

After first upload, re-import the cloud version to capture assigned `_postman_id` UUIDs:

```bash
curl -s "https://api.getpostman.com/collections/$COLLECTION_UID" \
    -H "X-Api-Key: $POSTMAN_API_KEY" \
    | jq '.collection' > collection.json
```

Future uploads will match items by UUID, preventing duplicates. Store `$POSTMAN_API_KEY` as an environment variable — never commit it.

## Anti-Patterns

| Anti-Pattern | Do Instead |
|-------------|------------|
| Hardcoded IDs in request bodies | Use `{{variable}}` references with chaining |
| Secrets in collection variables | Use environment variables for secrets |
| No delay in Newman runs | Use `--delay-request 100`+ to avoid rate limiter failures |
| Same token for all API layers | Separate token variables per layer |
| Manual editing in Postman cloud | Always edit locally, sync to cloud |
| Missing response examples | Include success + error examples for documentation |
| Not sending `Accept: application/json` | Laravel returns HTML on errors — set at collection level |
| `POSTMAN_API_KEY` in collection variables | Inject via CI secret: `--env-var "postman_api_key=$SECRET"` |
| `sleep` in CI to wait for app startup | Use health-check loop: `until curl -s $URL/health; do sleep 1; done` |
| No `.example` environment template committed | Commit `*.postman_environment.json.example` with `"value": "REPLACE_ME"` placeholders |

## Laravel-Specific Patterns

- **Form Request rules -> Body Parameters**: Parse validation rules to build parameter tables in request descriptions
- **API Resource wrapping**: All responses use `{ "data": ... }` wrapper — match in test assertions and examples
- **Rate limiter awareness**: Add `--delay-request` in Newman, include 429 examples in collections
- **422 validation format**: Laravel returns `{ "message": "...", "errors": { "field": ["..."] } }` — include these for endpoints with request bodies
