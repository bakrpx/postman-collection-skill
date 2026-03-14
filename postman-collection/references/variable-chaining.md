# Variable Chaining & Environment Management

Patterns for capturing data across requests, managing environment configurations, and implementing data-driven testing with Newman.

## Path Variable Resolution

For parameterized URLs like `/users/{userId}`, inject pre-request scripts to resolve path variables from collection variables:

```javascript
pm.request.url.variables.upsert({
    key: 'userId',
    value: pm.collectionVariables.get('user_id')
});
```

Map URL param names (camelCase) to collection variable names (snake_case). Add lookup pre-requests when a dependent resource may not exist:

```javascript
const _base = pm.variables.get('base_url');
const _token = pm.collectionVariables.get('access_token');
pm.sendRequest({
    url: _base + '/api/v1/users?per_page=1',
    method: 'GET',
    header: { 'Accept': 'application/json', 'Authorization': 'Bearer ' + _token }
}, (err, res) => {
    if (!err && res.code === 200) {
        const items = res.json().data;
        if (Array.isArray(items) && items.length > 0) {
            pm.collectionVariables.set('user_id', items[0].id);
        }
    }
});
```

## Environment File Management

### `.gitignore` Pattern

Store actual values locally only:

```
*.postman_environment.json
!*.postman_environment.json.example
```

### Example Template

Commit a `.example` version with placeholders:

```json
{
  "name": "Local",
  "values": [
    { "key": "base_url", "value": "REPLACE_ME", "enabled": true },
    { "key": "oauth_username", "value": "REPLACE_ME", "enabled": true },
    { "key": "oauth_password", "value": "REPLACE_ME", "enabled": true }
  ]
}
```

Developers copy this and replace `REPLACE_ME` with actual values for their environment.

### Multi-Environment Promotion

Values change per environment; structure stays constant:

| Environment | base_url | oauth_username | oauth_password |
|---|---|---|---|
| Local | `http://myapp.test` | `admin@local` | `password` |
| Dev | `https://dev.myapp.com` | `admin@dev` | `${DEV_PASS}` |
| Staging | `https://staging.myapp.com` | `admin@staging` | `${STAGING_PASS}` |
| Production | `https://api.myapp.com` | `admin@prod` | `${PROD_PASS}` |

Key names (`base_url`, `oauth_username`) never change — only values. Tests and scripts reference the same variables regardless of environment.

## Data-Driven Testing with Newman

Run a collection repeatedly with different input data (CSV or JSON):

```bash
npx newman run collection.json -e environments/Local.json \
    --iteration-data data/users.csv --delay-request 100
```

### CSV Format

```
email,firstName,lastName
alice@example.com,Alice,Smith
bob@example.com,Bob,Jones
charlie@example.com,Charlie,Brown
```

### JSON Format

```json
[
  { "email": "alice@example.com", "firstName": "Alice", "lastName": "Smith" },
  { "email": "bob@example.com", "firstName": "Bob", "lastName": "Jones" }
]
```

### Accessing Iteration Data in Scripts

```javascript
// In pre-request or test scripts
const email = pm.iterationData.get('email');
pm.collectionVariables.set('_email', email);
```

### Request Body with Variables

```json
{
  "email": "{{email}}",
  "firstName": "{{firstName}}",
  "lastName": "{{lastName}}"
}
```

Each iteration runs the full collection with values from one row, enabling bulk testing, multi-user scenarios, and load simulation.
