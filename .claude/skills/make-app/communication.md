# Communication — request & response

`communication.imljson` files describe one HTTP request, or an array of requests (executed sequentially; last one's output becomes the module output).

## Request-level keys

| Key                  | Type                  | Default | Purpose                                                       |
| -------------------- | --------------------- | ------- | ------------------------------------------------------------- |
| `url`                | IML string (≤512)     | —       | Absolute URL or relative path appended to `base.baseUrl`.     |
| `method`             | enum or IML           | `GET`   | **Only** `GET` / `POST` / `PUT` / `PATCH` / `DELETE`.         |
| `headers`            | flat IML object       | —       | Headers (case-insensitive). Merged with base headers.         |
| `qs`                 | flat IML object       | —       | Query string. Arrays expand: `["a","b"]` → `&k=a&k=b`.        |
| `body`               | any IML               | —       | Payload. **Required when method is POST/PUT/PATCH.**          |
| `type`               | enum or IML           | `json`  | Request body encoding (see below).                            |
| `encodeUrl`          | boolean               | `true`  | If false, Make won't URL-encode `{{...}}` substitutions.      |
| `condition`          | bool/string or object | `true`  | Skip request when false. Object form: `{condition, default}`. |
| `ca`                 | string (≤8192)        | —       | Custom certificate authority PEM.                             |
| `aws`                | object                | —       | AWS signing helper (see below).                               |
| `oauth`              | object                | —       | OAuth-1 signing helper.                                       |
| `log`                | object                | —       | `{ "sanitize": [...] }`.                                      |
| `gzip`               | boolean               | `false` | Send/accept gzip (api.json only — not base).                  |
| `followRedirects`    | boolean               | `true`  | Follow `3xx` redirects for `GET`.                             |
| `followAllRedirects` | boolean               | `true`  | Follow `3xx` redirects for non-`GET`.                         |
| `pagination`         | object                | —       | See `pagination.md`.                                          |
| `temp`               | object                | —       | Set temp vars BEFORE this request runs.                       |
| `repeat`             | object                | —       | `{ condition, delay (ms), limit }` — retry the request.       |
| `iterate`            | string \| object      | —       | Webhook-only: loop over batched events.                       |
| `output`             | any                   | —       | Webhook-only: bundle shape.                                   |
| `respond`            | object                | —       | Webhook-only: custom HTTP response.                           |
| `verification`       | object                | —       | Webhook-only: challenge handshake.                            |
| `response`           | object                | —       | Response handling (below).                                    |

### `type` enum (request body encoding)
`json` (default) · `urlencoded` · `multipart/form-data` · `text` · `string` · `raw` · `binary`

### `condition` object form (return default instead of skipping)
```json
"condition": {
  "condition": "{{parameters.skipEnrichment}}",
  "default":   { "skipped": true }
}
```

## `response` keys

| Key         | Type                       | Purpose                                                    |
| ----------- | -------------------------- | ---------------------------------------------------------- |
| `type`      | string / status-keyed obj  | Parse type (default: auto from Content-Type).              |
| `valid`     | IML string \| object       | Override success/fail judgment.                            |
| `error`     | object                     | Error spec (see `errors.md`).                              |
| `limit`     | IML number/string          | Max items in output (Search/Trigger).                      |
| `iterate`   | string \| object           | Loop over an array → one bundle per item.                  |
| `output`    | any                        | Shape of each emitted bundle.                              |
| `wrapper`   | object                     | Wraps final array under top-level extras (runs once).      |
| `temp`      | object                     | Save data to `temp.*` after this request.                  |
| `data`      | object                     | Persist into module's `data.*` (triggers, attach).         |
| `trigger`   | object                     | Polling trigger config (`type`, `order`, `id`, `date`).    |
| `uid`       | IML string                 | Shared-webhook user routing.                               |
| `metadata`  | object                     | Connection display label.                                  |

## Basic single-request example
```json
{
  "url": "/users/{{parameters.id}}",
  "method": "GET",
  "response": { "output": "{{body}}" }
}
```

## Multiple requests (array)
```json
[
  {
    "url": "/me",
    "method": "GET",
    "response": { "temp": { "username": "{{body.username}}" } }
  },
  {
    "url": "/items",
    "method": "GET",
    "response": {
      "iterate": "{{body.items}}",
      "output": {
        "username": "{{temp.username}}",
        "title":    "{{item.title}}"
      }
    }
  }
]
```

## `iterate`

Simple form:
```json
"iterate": "{{body.data}}"
```

Filtered form:
```json
"iterate": {
  "container": "{{body.data}}",
  "condition": "{{item.active == true}}"
}
```

Inside `output`, the current array element is `item`.

## `output`

Defaults:
- No `iterate` → output = `body`.
- With `iterate` → output = `item`.

Shaped output:
```json
"output": {
  "id":         "{{item.id}}",
  "title":      "{{item.subject}}",
  "created_at": "{{parseDate(item.created, 'YYYY-MM-DD HH:mm:ss')}}"
}
```

Pass through a custom IML function:
```json
"output": "{{parseEntity(body)}}"
```

## `wrapper`

Adds top-level keys around the iterated array. `output` inside `wrapper` refers to the transformed iteration result.

```json
"iterate": "{{body.users}}",
"output": { "label": "{{item.name}}", "value": "{{item.id}}" },
"wrapper": {
  "type":  "select",
  "data":  "{{output}}",
  "total": "{{body.total_count}}"
}
```

## `limit`

Caps emitted items. For multi-request chains, only the final request's `limit` matters.

```json
"response": { "limit": "{{parameters.limit}}" }
```

Always respect the user-provided `limit` parameter.

## `temp` — cross-request memory

Lives for the module execution. Useful for cursor pagination, IDs fetched in step 1 used in step 2.

```json
"response": {
  "temp": { "username": "{{body.username}}", "session_id": "{{body.session.id}}" }
}
```

Use later: `{{temp.username}}`.

## `data` — persisted module state

Lives across executions of the same module instance. Use for trigger cursors (`data.lastDate`/`data.lastID` are auto-managed) and webhook attach IDs (`data.externalHookId`).

```json
"response": {
  "data": { "externalHookId": "{{body.id}}", "token": "{{body.token}}" }
}
```

## `valid` — body-level error detection

Some APIs return 200 OK with `error` in body.

Simple:
```json
"valid": "{{!body.error}}"
```

Full:
```json
"valid": {
  "condition": "{{body.status != 'error'}}",
  "message":   "{{body.error_description}}",
  "type":      "DataError"
}
```

See `errors.md` for error spec.

## `type` parsing

Default: auto from response `Content-Type`. Override:
```json
"response": { "type": "json" }
```

Per-status parsing:
```json
"response": {
  "type": { "*": "json", "400-408": "text", "406": "xml" }
}
```

XML quirks:
- All nodes are arrays even for single elements.
- Mixed content nodes: text in `_value`, attributes in `_attributes`.

Numbers larger than `Number.MAX_SAFE_INTEGER` — use `type: "text"` and reparse with regex + `parseJSON`.

## Content types (`type` on the request)

| `type`               | Content-Type                       | Body shape                                             |
| -------------------- | ---------------------------------- | ------------------------------------------------------ |
| `json` (default)     | `application/json`                 | JSON object / array                                    |
| `urlencoded`         | `application/x-www-form-urlencoded`| flat object                                            |
| `multipart/form-data`| `multipart/form-data`              | object; for files use `{ value, options: { filename } }` |
| `text`               | `text/plain`                       | string                                                 |
| `string`             | as-is                              | string (no parsing)                                    |
| `binary`             | `application/octet-stream`         | Buffer                                                 |
| `raw`                | leave to user                      | string sent as-is                                      |

> Note: webhook `respond.type` is restricted to `json` / `urlencoded` / `text` only.

### multipart/form-data
```json
{
  "url": "/files",
  "method": "POST",
  "type": "multipart/form-data",
  "body": {
    "file": {
      "value": "{{parameters.fileData}}",
      "options": { "filename": "{{parameters.fileName}}" }
    },
    "title": "{{parameters.title}}"
  }
}
```

Use `semantic: "file:data"` (buffer) + `semantic: "file:name"` (filename) on the params.

## Request-less communication

Omit `url` for purely computed output:
```json
{
  "response": {
    "output": {
      "summary": "Item {{parameters.id}}: {{parameters.text}}"
    }
  }
}
```

## `repeat` — automatic retries
```json
"repeat": {
  "condition": "{{statusCode == 503}}",
  "delay":     2000,
  "limit":     3
}
```

Use for transient errors. For 429s prefer `RateLimitError` in `response.error` (built-in exponential backoff).

## `aws.signature`

AWS request signing.

| Field          | Meaning                                             |
| -------------- | --------------------------------------------------- |
| `key`          | Access key ID                                       |
| `secret`       | Secret key                                          |
| `session`      | Session token (when relevant)                       |
| `bucket`       | S3 bucket (when not in URL)                         |
| `sign_version` | `"2"` (default) or `"4"`                            |
| `service`      | Required with v4 (e.g. `"s3"`, `"sqs"`)             |
| `region`       | Required with v4                                    |

```json
{
  "url": "https://example.amazonaws.com",
  "aws": {
    "key":          "{{connection.key}}",
    "secret":       "{{connection.secret}}",
    "sign_version": "4",
    "service":      "s3",
    "region":       "us-east-1"
  }
}
```

Not available inside OAuth 1/2 connection communication.

## JSON-strings inside JSON

Parse string → object:
```json
"address": "{{parseJSON(body.address)}}"
```

Object → string:
```json
"address": "{{createJSON(parameters.address)}}"
```

## Buffer (file) transit

Download modules return `{ name: "...", data: <buffer> }`. Upload modules accept the same shape — values flow automatically between Make modules.

## `log.sanitize`

```json
"log": {
  "sanitize": [
    "request.headers.authorization",
    "request.body.password",
    "request.qs.api_key",
    "response.body.access_token"
  ]
}
```

Without `log.sanitize`, Make hides logs entirely from DevTool.

## Source URLs

- https://developers.make.com/custom-apps-documentation/component-blocks/api.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/making-requests.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/multiple-requests.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/multipart-form-data.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/handling-responses.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/handling-responses/iterate.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/handling-responses/output.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/handling-responses/valid.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/handling-responses/limit.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/handling-responses/type.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/handling-responses/temp.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/request-less-communication.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/processing-of-json-strings-inside-a-json-object.md
