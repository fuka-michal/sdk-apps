# Base — `base.imljson`

The Base is a single JSON object holding HTTP defaults inherited by **every module and RPC** (unless they override). After app approval, Base is locked.

## Top-level fields (base schema)

| Field                | Type                  | Purpose                                                  |
| -------------------- | --------------------- | -------------------------------------------------------- |
| `baseUrl`            | string (≤512)         | Root API URL. Module `url` values are appended.          |
| `url`                | string (≤512)         | Optional override; rarely used at base level.            |
| `headers`            | flat IML object       | Default request headers (merged into every request).     |
| `qs`                 | flat IML object       | Default query-string params.                             |
| `body`               | any IML               | Default request body. **Required if `method` is POST/PUT/PATCH/DELETE.** |
| `method`             | enum or IML           | `GET`/`POST`/`PUT`/`PATCH`/`DELETE`.                     |
| `type`               | request-type enum     | Default request body encoding.                           |
| `encodeUrl`          | boolean (default true)| Auto-encode `{{...}}` in URL.                            |
| `temp`               | object                | Set temp vars before requests.                           |
| `condition`          | bool/string or object | Skip request when false.                                 |
| `ca`                 | string                | Custom CA PEM (≤8192).                                   |
| `aws`                | object                | AWS signing helper.                                      |
| `oauth`              | object                | OAuth-1 signing helper (rare).                           |
| `pagination`         | object                | Default pagination config.                               |
| `followRedirects`    | boolean (default true)| Follow 3xx redirects for GET.                            |
| `followAllRedirects` | boolean (default true)| Follow 3xx redirects for non-GET.                        |
| `log`                | `{ sanitize: [...] }` | Dot-paths to redact in execution logs.                   |
| `response`           | object                | Response handling defaults (see below).                  |

### Response sub-directives in base
`type` · `valid` · `limit` · `error` · `iterate` · `temp` · `output` · `data` · `metadata` · `uid` · `oauth` · `wrapper`

### NOT available in base (api.json only)
- `gzip` — only on individual requests
- `respond` — webhook-only
- `verification` — webhook-only
- `repeat` — only on individual requests
- Request-level `iterate` / `output` — only on individual requests
- `response.trigger` — module-level polling trigger config
- `response.expires` — connection refresh config

## Example — OAuth2 Bearer + global error spec

```json
{
  "baseUrl": "https://api.github.com",
  "headers": {
    "authorization": "Bearer {{connection.accessToken}}",
    "accept": "application/vnd.github+json",
    "user-agent": "make-app",
    "x-github-api-version": "2022-11-28"
  },
  "response": {
    "error": {
      "message": "[{{statusCode}}] {{body.message}}",
      "401": { "type": "InvalidAccessTokenError", "message": "Access token is invalid or expired" },
      "403": { "type": "RuntimeError", "message": "{{body.message}}" },
      "404": { "type": "DataError", "message": "Resource not found: {{body.message}}" },
      "422": { "type": "DataError", "message": "{{body.message}}" },
      "429": { "type": "RateLimitError", "message": "{{body.message}}" },
      "500": { "type": "ConnectionError", "message": "GitHub server error: {{body.message}}" }
    }
  },
  "log": {
    "sanitize": ["request.headers.authorization"]
  }
}
```

## Authorization patterns

API key in header:
```json
{ "headers": { "x-api-key": "{{connection.apiKey}}" } }
```

API key in query string:
```json
{ "qs": { "apikey": "{{connection.apiKey}}" } }
```

OAuth2 bearer:
```json
{ "headers": { "authorization": "Bearer {{connection.accessToken}}" } }
```

Basic auth:
```json
{ "headers": { "authorization": "Basic {{base64(connection.username + ':' + connection.password)}}" } }
```

JWT (built dynamically):
```json
{ "headers": { "authorization": "Bearer {{jwt({iss: connection.email, scope: 'foo', aud: 'bar', exp: addMinutes(now, 30), iat: now}, connection.privateKey, 'RS256')}}" } }
```

## Inheritance & overrides

- A module-level key **fully replaces** the base for that key. To merge with base instead, use spread syntax:
  ```json
  { "headers": { "{{...}}": "{{buildExtraHeaders()}}" } }
  ```
- Most apps never override the base; modules just set `url`, `method`, `qs`, `body`.

## `response` defaults inherited by modules

```json
{
  "response": {
    "valid": "{{!body.error}}",          // marks a 2xx with body-error as failure
    "error": {
      "type": "RuntimeError",
      "message": "[{{statusCode}}] {{body.error.message}}"
    },
    "type": "json"
  }
}
```

A module may add `iterate` / `output` / `pagination` without redefining these.

## `log.sanitize` rules

- Array of dot-notation paths.
- Prefix `request.` or `response.`.
- Header names with hyphens use backticks: ``"request.headers.`X-API-Key`"``
- Common patterns to always sanitize:
  - `request.headers.authorization`
  - `request.qs.api_key`, `request.qs.access_token`
  - `request.body.password`, `request.body.client_secret`, `request.body.code`
  - `response.body.access_token`, `response.body.refresh_token`

**Without `log.sanitize`, execution logs are entirely hidden in Make DevTool** and secrets may still be persisted server-side. Sanitize first; observability follows.

## Source URLs

- https://developers.make.com/custom-apps-documentation/app-components/base.md
- https://developers.make.com/custom-apps-documentation/app-components/base/authorization.md
- https://developers.make.com/custom-apps-documentation/app-components/base/error-handling.md
- https://developers.make.com/custom-apps-documentation/app-components/base/sanitization.md
- https://developers.make.com/custom-apps-documentation/app-components/base/advanced-inheritance.md
