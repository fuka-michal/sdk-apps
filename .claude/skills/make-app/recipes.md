# Recipes — Copy-paste starters

Drop-in templates for the most common module / connection / webhook / RPC patterns. Customize names and endpoints.

## Connection — OAuth 2.0

`connections/<name>/metadata.json`:
```json
{ "label": "Service OAuth Connection", "type": "oauth" }
```

`parameters.imljson`:
```json
[
  { "name": "clientId",     "type": "text",     "label": "Client ID",     "help": "Your service's Client ID." },
  { "name": "clientSecret", "type": "password", "label": "Client Secret", "help": "Your service's Client Secret." },
  { "name": "scopes",       "type": "array",    "label": "Additional scopes", "spec": { "type": "text" } }
]
```

`scope.imljson`:
```json
["read:user"]
```

`scopes.imljson`:
```json
{ "read:user": "Read the authenticated user's profile" }
```

`communication.imljson`:
```json
{
  "authorize": {
    "url": "https://service.com/oauth/authorize",
    "qs": {
      "client_id":     "{{ifempty(parameters.clientId, common.clientId)}}",
      "redirect_uri":  "{{oauth.redirectUri}}",
      "scope":         "{{join(distinct(merge(oauth.scope, ifempty(parameters.scopes, emptyarray))), ' ')}}",
      "response_type": "code"
    },
    "response": { "temp": { "code": "{{query.code}}" } }
  },
  "token": {
    "url":    "https://service.com/oauth/token",
    "method": "POST",
    "type":   "urlencoded",
    "body": {
      "code":          "{{temp.code}}",
      "client_id":     "{{ifempty(parameters.clientId, common.clientId)}}",
      "client_secret": "{{ifempty(parameters.clientSecret, common.clientSecret)}}",
      "redirect_uri":  "{{oauth.redirectUri}}",
      "grant_type":    "authorization_code"
    },
    "response": {
      "data": {
        "accessToken":  "{{body.access_token}}",
        "refreshToken": "{{body.refresh_token}}",
        "expires":      "{{addSeconds(now, body.expires_in)}}"
      }
    },
    "log": { "sanitize": ["request.body.code", "request.body.client_secret", "response.body.access_token", "response.body.refresh_token"] }
  },
  "info": {
    "url":     "https://api.service.com/me",
    "headers": { "authorization": "Bearer {{connection.accessToken}}" },
    "response": {
      "uid":      "{{body.id}}",
      "metadata": { "type": "email", "value": "{{body.email}}" }
    },
    "log": { "sanitize": ["request.headers.authorization"] }
  },
  "refresh": {
    "condition": "{{data.expires < addMinutes(now, 15)}}",
    "url":       "https://service.com/oauth/token",
    "method":    "POST",
    "type":      "urlencoded",
    "body": {
      "grant_type":    "refresh_token",
      "refresh_token": "{{data.refreshToken}}",
      "client_id":     "{{ifempty(parameters.clientId, common.clientId)}}",
      "client_secret": "{{ifempty(parameters.clientSecret, common.clientSecret)}}"
    },
    "response": {
      "data": {
        "accessToken":  "{{body.access_token}}",
        "refreshToken": "{{body.refresh_token}}",
        "expires":      "{{addSeconds(now, body.expires_in)}}"
      }
    },
    "log": { "sanitize": ["request.body.refresh_token", "request.body.client_secret", "response.body.access_token", "response.body.refresh_token"] }
  }
}
```

## Connection — API Key (Basic)

`metadata.json`:
```json
{ "label": "Service Connection", "type": "basic" }
```

`parameters.imljson`:
```json
[
  { "name": "apiKey", "type": "password", "label": "API Key", "required": true,
    "help": "Find at https://service.com/settings/api" }
]
```

`communication.imljson`:
```json
{
  "url":     "https://api.service.com/me",
  "headers": { "x-api-key": "{{parameters.apiKey}}" },
  "response": {
    "metadata": { "type": "email", "value": "{{body.email}}" }
  },
  "log": { "sanitize": ["request.headers.`x-api-key`"] }
}
```

## Base — OAuth2 Bearer

```json
{
  "baseUrl": "https://api.service.com/v1",
  "headers": {
    "authorization": "Bearer {{connection.accessToken}}",
    "accept":        "application/json"
  },
  "response": {
    "error": {
      "message": "[{{statusCode}}] {{body.error.message}}",
      "401": { "type": "InvalidAccessTokenError", "message": "Token expired" },
      "403": { "type": "RuntimeError",            "message": "{{body.error.message}}" },
      "404": { "type": "DataError",               "message": "Not found: {{body.error.message}}" },
      "422": { "type": "DataError",               "message": "{{body.error.message}}" },
      "429": { "type": "RateLimitError",          "message": "{{body.error.message}}" },
      "500": { "type": "ConnectionError",         "message": "Server error" }
    }
  },
  "log": {
    "sanitize": ["request.headers.authorization"]
  }
}
```

## Action — Create record (typeId 4)

`metadata.json`:
```json
{
  "label": "Create a record",
  "description": "Creates a new record.",
  "typeId": 4,
  "crud": "create",
  "attachedAccounts": ["<connection-name>"]
}
```

`communication.imljson`:
```json
{
  "url":    "/records",
  "method": "POST",
  "body": {
    "title":       "{{parameters.title}}",
    "description": "{{parameters.description}}",
    "tags":        "{{parameters.tags}}"
  },
  "response": { "output": "{{body}}" }
}
```

`expect.imljson`:
```json
[
  { "name": "title",       "type": "text",  "label": "Title", "required": true },
  { "name": "description", "type": "text",  "label": "Description", "multiline": true },
  { "name": "tags",        "type": "array", "label": "Tags", "spec": { "type": "text" } }
]
```

## Search — List records (typeId 9)

`metadata.json`:
```json
{
  "label": "List records",
  "description": "Returns a list of records.",
  "typeId": 9,
  "crud": "read",
  "attachedAccounts": ["<connection-name>"]
}
```

`communication.imljson`:
```json
{
  "url":    "/records",
  "method": "GET",
  "qs": {
    "q":        "{{parameters.query}}",
    "sort":     "{{parameters.sort}}",
    "per_page": "{{ifempty(parameters.limit, 30)}}"
  },
  "pagination": {
    "qs":        { "page": "{{pagination.page}}" },
    "condition": "{{length(body.data) > 0}}"
  },
  "response": {
    "iterate": "{{body.data}}",
    "output":  "{{item}}",
    "limit":   "{{parameters.limit}}"
  }
}
```

`expect.imljson`:
```json
[
  { "name": "query", "type": "text",   "label": "Query" },
  { "name": "sort",  "type": "select", "label": "Sort",
    "options": [
      { "label": "Created (newest)", "value": "-created" },
      { "label": "Created (oldest)", "value": "created"  }
    ]
  },
  { "name": "limit", "type": "uinteger", "label": "Limit", "default": 10 }
]
```

## Polling Trigger — Watch records (typeId 1)

`metadata.json`:
```json
{
  "label": "Watch records",
  "description": "Triggers when a new record is created.",
  "typeId": 1,
  "crud": "read",
  "attachedAccounts": ["<connection-name>"]
}
```

`communication.imljson`:
```json
{
  "url":    "/records",
  "method": "GET",
  "qs": {
    "sort":      "created",
    "direction": "desc",
    "since":     "{{data.lastDate}}",
    "per_page":  "{{ifempty(parameters.limit, 100)}}"
  },
  "pagination": { "qs": { "page": "{{pagination.page}}" } },
  "response": {
    "iterate": "{{body.data}}",
    "trigger": {
      "type":  "date",
      "order": "desc",
      "id":    "{{item.id}}",
      "date":  "{{item.created_at}}"
    },
    "output": "{{item}}",
    "limit":  "{{parameters.limit}}"
  }
}
```

`parameters.imljson`:
```json
[
  { "name": "limit", "type": "uinteger", "label": "Limit", "default": 10, "required": true }
]
```

`epoch.imljson`:
```json
{
  "url": "/records",
  "qs":  { "sort": "created", "direction": "desc", "per_page": 100 },
  "response": {
    "iterate": "{{body.data}}",
    "output":  { "id": "{{item.id}}", "date": "{{item.created_at}}", "label": "{{item.title}}" },
    "limit":   300
  }
}
```

## Instant Trigger + Dedicated Webhook (typeId 10)

`modules/watchRecordsInstant/metadata.json`:
```json
{
  "label": "Watch records (instant)",
  "description": "Triggers immediately when a record event occurs.",
  "typeId":  10,
  "crud":    "read",
  "webhook": "recordsWebhook"
}
```

`webhooks/recordsWebhook/metadata.json`:
```json
{ "label": "Records webhook", "type": "web" }
```

`webhooks/recordsWebhook/attach.imljson`:
```json
{
  "url":    "/webhooks",
  "method": "POST",
  "body": {
    "url":    "{{webhook.url}}",
    "events": "{{parameters.events}}"
  },
  "response": {
    "data": { "externalHookId": "{{body.id}}" }
  }
}
```

`webhooks/recordsWebhook/detach.imljson`:
```json
{
  "url":    "/webhooks/{{data.externalHookId}}",
  "method": "DELETE"
}
```

`webhooks/recordsWebhook/communication.imljson`:
```json
{
  "verification": {
    "condition": "{{body.challenge}}",
    "respond":   { "type": "text", "body": "{{body.challenge}}" }
  },
  "output": "{{body}}"
}
```

`webhooks/recordsWebhook/parameters.imljson`:
```json
[
  { "name": "events", "type": "array", "label": "Events", "required": true,
    "spec": { "type": "select", "options": [
      { "label": "Created", "value": "record.created" },
      { "label": "Updated", "value": "record.updated" },
      { "label": "Deleted", "value": "record.deleted" }
    ]}
  }
]
```

## Universal Module (typeId 12)

`modules/makeAPICall/metadata.json`:
```json
{
  "label": "Make an API call",
  "description": "Sends a custom API call to Service. You can use this to call endpoints that aren't covered by existing modules",
  "typeId": 12,
  "crud":   "read",
  "attachedAccounts": ["<connection-name>"]
}
```

`communication.imljson`:
```json
{
  "url":     "{{parameters.url}}",
  "method":  "{{parameters.method}}",
  "qs":      "{{parameters.qs}}",
  "headers": "{{parameters.headers}}",
  "body":    "{{parameters.body}}",
  "response": {
    "output": {
      "statusCode": "{{statusCode}}",
      "headers":    "{{headers}}",
      "body":       "{{body}}"
    }
  }
}
```

`expect.imljson`:
```json
[
  { "name": "url",    "type": "text",   "label": "URL", "required": true,
    "help": "Relative path, e.g. /records/123" },
  { "name": "method", "type": "select", "label": "Method", "required": true, "default": "GET",
    "options": [
      { "label": "GET", "value": "GET" }, { "label": "POST", "value": "POST" },
      { "label": "PUT", "value": "PUT" }, { "label": "PATCH", "value": "PATCH" },
      { "label": "DELETE", "value": "DELETE" }
    ]
  },
  { "name": "qs", "type": "array", "label": "Query String", "spec": [
    { "name": "key", "type": "text" }, { "name": "value", "type": "text" }
  ]},
  { "name": "headers", "type": "array", "label": "Headers", "spec": [
    { "name": "key", "type": "text" }, { "name": "value", "type": "text" }
  ]},
  { "name": "body", "type": "any", "label": "Body" }
]
```

## RPC — Dynamic Options

`rpc/listRecords/metadata.json`:
```json
{ "label": "List records" }
```

`rpc/listRecords/communication.imljson`:
```json
{
  "url": "/records",
  "qs":  { "per_page": 100 },
  "pagination": {
    "qs":        { "page": "{{pagination.page}}" },
    "condition": "{{pagination.page <= 3}}"
  },
  "response": {
    "iterate": "{{body.data}}",
    "output":  { "label": "{{item.title}}", "value": "{{item.id}}" },
    "limit":   300
  }
}
```

Use:
```json
{ "name": "recordId", "type": "select", "label": "Record",
  "options": "rpc://listRecords", "required": true }
```

## RPC — Dynamic Fields

`rpc/dynamicFields/communication.imljson`:
```json
{
  "url": "/schemas/{{parameters.entity}}",
  "response": {
    "iterate": "{{body.fields}}",
    "output": {
      "name":     "{{item.name}}",
      "label":    "{{item.label}}",
      "type":     "{{item.type}}",
      "required": "{{item.required}}"
    }
  }
}
```

Use:
```json
[
  { "name": "entity", "type": "select", "label": "Entity",
    "options": [{ "label": "Contact", "value": "contact" }, { "label": "Deal", "value": "deal" }] },
  "rpc://dynamicFields?entity={{parameters.entity}}"
]
```

## Multipart file upload

`expect.imljson`:
```json
[
  { "name": "fileName", "type": "filename", "label": "File name", "semantic": "file:name", "required": true },
  { "name": "fileData", "type": "buffer",   "label": "File data", "semantic": "file:data", "required": true },
  { "name": "title",    "type": "text",     "label": "Title" }
]
```

`communication.imljson`:
```json
{
  "url":    "/files",
  "method": "POST",
  "type":   "multipart/form-data",
  "body": {
    "file": {
      "value":   "{{parameters.fileData}}",
      "options": { "filename": "{{parameters.fileName}}" }
    },
    "title": "{{parameters.title}}"
  },
  "response": { "output": "{{body}}" }
}
```

## Custom IML function — `removeEmpty`

`functions/removeEmpty/code.js`:
```javascript
function removeEmpty(obj) {
    if (Array.isArray(obj)) {
        return obj.map(removeEmpty).filter(v => v !== undefined && v !== null);
    }
    if (obj && typeof obj === 'object') {
        const out = {};
        for (const k of Object.keys(obj)) {
            const v = removeEmpty(obj[k]);
            if (v === undefined || v === null) continue;
            if (typeof v === 'string' && v === '') continue;
            if (Array.isArray(v) && v.length === 0) continue;
            if (typeof v === 'object' && !Array.isArray(v) && Object.keys(v).length === 0) continue;
            out[k] = v;
        }
        return out;
    }
    return obj;
}
```

`functions/removeEmpty/test/test.js`:
```javascript
it('removes null, undefined, empty strings, empty arrays/objects', () => {
    const input  = { a: 1, b: null, c: '', d: [], e: {}, f: { g: undefined, h: 'x' } };
    const expect = { a: 1, f: { h: 'x' } };
    assert.deepStrictEqual(removeEmpty(input), expect);
});
```

Use in body:
```json
{ "body": "{{removeEmpty(parameters)}}" }
```
