# Auth Patterns Reference

Authentication and authorization patterns for Postman collections serving multi-layer Laravel APIs.

## Auth Architecture Overview

Most Laravel APIs have multiple authentication contexts. Each API layer may use a different auth mechanism:

| Layer | Auth Method | Token Variable | Header |
|-------|------------|----------------|--------|
| Admin | JWT (OAuth2) | `admin_access_token` | `Authorization: Bearer {{token}}` |
| Tenant | JWT (direct or delegated) | `tenant_access_token` | `Authorization: Bearer {{token}}` + `X-Tenant-Id: {{tenant_id}}` |
| Public | API Key | `api_key` | `X-API-Key: {{api_key}}` |
| User Auth | Sanctum/Passport | `user_token` | `Authorization: Bearer {{token}}` |

## JWT Token Acquisition

### OAuth2 Password Grant

Pre-request script for the collection or auth folder:

```javascript
// Token caching — skip if still valid
const expiry = parseInt(pm.collectionVariables.get('token_expiry') || '0');
if (Date.now() / 1000 < expiry - 30) return;

const oauthUrl = pm.variables.get('oauth_url');

pm.sendRequest({
    url: `${oauthUrl}/oauth/token`,
    method: 'POST',
    header: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: {
        mode: 'urlencoded',
        urlencoded: [
            { key: 'grant_type', value: 'password' },
            { key: 'client_id', value: pm.variables.get('oauth_client_id') },
            { key: 'client_secret', value: pm.variables.get('oauth_client_secret') },
            { key: 'username', value: pm.variables.get('oauth_username') },
            { key: 'password', value: pm.variables.get('oauth_password') },
        ]
    }
}, (err, res) => {
    if (err || res.code !== 200) {
        console.error('Token fetch failed:', err, res?.code);
        return;
    }
    const body = res.json();
    pm.collectionVariables.set('access_token', body.access_token);
    pm.collectionVariables.set('admin_access_token', body.access_token);
    pm.collectionVariables.set('token_expiry',
        String(Math.floor(Date.now() / 1000) + body.expires_in)
    );
});
```

### Token Refresh

For long-running Newman suites, refresh tokens before they expire:

```javascript
const expiry = parseInt(pm.collectionVariables.get('token_expiry') || '0');
const now = Math.floor(Date.now() / 1000);

if (now < expiry - 60) return; // Still valid

const refreshToken = pm.collectionVariables.get('refresh_token');
if (!refreshToken) return;

const oauthUrl = pm.variables.get('oauth_url');

pm.sendRequest({
    url: `${oauthUrl}/oauth/token`,
    method: 'POST',
    header: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: {
        mode: 'urlencoded',
        urlencoded: [
            { key: 'grant_type', value: 'refresh_token' },
            { key: 'client_id', value: pm.variables.get('oauth_client_id') },
            { key: 'refresh_token', value: refreshToken },
        ]
    }
}, (err, res) => {
    if (err || res.code !== 200) return;
    const body = res.json();
    pm.collectionVariables.set('access_token', body.access_token);
    pm.collectionVariables.set('refresh_token', body.refresh_token);
    pm.collectionVariables.set('token_expiry',
        String(Math.floor(Date.now() / 1000) + body.expires_in)
    );
});
```

## Laravel Sanctum Token Flow

### Token Creation (Login)

```javascript
// Test script: capture token from login response
pm.test('Login successful', () => {
    pm.response.to.have.status(200);
    const body = pm.response.json();
    pm.collectionVariables.set('user_token', body.token || body.access_token);
});
```

### Sanctum Cookie-Based Auth (SPA)

```javascript
// Step 1: Get CSRF cookie
pm.sendRequest({
    url: pm.variables.get('base_url') + '/sanctum/csrf-cookie',
    method: 'GET',
}, (err, res) => {
    // Cookie is automatically stored by Postman
});

// Step 2: Login with credentials (cookies handled automatically)
// The X-XSRF-TOKEN header is set from the cookie
```

## API Key Authentication

### Header-Based API Key

Collection auth configuration:

```json
{
  "auth": {
    "type": "apikey",
    "apikey": [
      { "key": "key", "value": "X-API-Key", "type": "string" },
      { "key": "value", "value": "{{api_key}}", "type": "string" },
      { "key": "in", "value": "header", "type": "string" }
    ]
  }
}
```

### Dynamic API Key from Login

```javascript
if (pm.response.code === 201) {
    const data = pm.response.json().data;
    if (data.api_key) {
        pm.collectionVariables.set('api_key', data.api_key);
    }
}
```

## `noauth` Inheritance Pattern

Set collection-level auth to `noauth` and configure auth per folder:

```json
{
  "auth": { "type": "noauth" },
  "item": [
    {
      "name": "Admin API",
      "auth": {
        "type": "bearer",
        "bearer": [{ "key": "token", "value": "{{admin_access_token}}" }]
      },
      "item": []
    },
    {
      "name": "Public API",
      "auth": {
        "type": "apikey",
        "apikey": [
          { "key": "key", "value": "X-API-Key" },
          { "key": "value", "value": "{{api_key}}" },
          { "key": "in", "value": "header" }
        ]
      },
      "item": []
    }
  ]
}
```

This prevents accidental token leakage between API layers.

## Multi-Role Token Storage

Use separate variables per API layer:

```javascript
// Admin token (from OAuth2)
pm.collectionVariables.set('admin_access_token', token);
pm.collectionVariables.set('admin_token_expiry', expiry);

// Tenant token (from direct login or delegation)
pm.collectionVariables.set('tenant_access_token', token);
pm.collectionVariables.set('tenant_token_expiry', expiry);
pm.collectionVariables.set('tenant_id', tenantId);

// User token (from Sanctum login)
pm.collectionVariables.set('user_token', token);

// API key (from API client creation)
pm.collectionVariables.set('api_key', apiKey);
```

Each folder uses its own token variable in its auth config, so Newman automatically sends the correct credentials.

## Environment Variables for Auth

Store secrets in environment files, never in collection variables:

```json
{
  "name": "Local",
  "values": [
    { "key": "base_url", "value": "https://myapp.test", "enabled": true },
    { "key": "oauth_url", "value": "https://myapp.test", "enabled": true },
    { "key": "oauth_client_id", "value": "my-client", "enabled": true },
    { "key": "oauth_client_secret", "value": "secret", "enabled": true },
    { "key": "oauth_username", "value": "admin@example.com", "enabled": true },
    { "key": "oauth_password", "value": "password", "enabled": true }
  ]
}
```

## Anti-Patterns

| Anti-Pattern | Risk | Do Instead |
|-------------|------|------------|
| Hardcoded tokens in collection | Tokens expire, collection becomes stale | Use pre-request token fetch with caching |
| Same token for all API layers | Permission leakage between layers | Separate token variables per layer |
| Secrets in collection variables | Leaked when sharing collection JSON | Use environment variables for secrets |
| No token expiry check | Unnecessary auth requests every call | Cache with expiry timestamp |
