# Error Handling

Configured under `response.error`. The default error type is `RuntimeError`. 4xx/5xx are automatically treated as errors.

## Error types (canonical 9)

These are the **only** valid `type` values — anything else is a schema violation.

| Type                          | Behaviour                                                                               |
| ----------------------------- | --------------------------------------------------------------------------------------- |
| `RuntimeError` (default)      | Interrupts execution, rolls back. Consumes the 3-consecutive-errors counter → eventually deactivates scenario scheduling. |
| `DataError`                   | Invalid incoming data; allows "incomplete execution" resume.                            |
| `ConnectionError`             | Network / 5xx issues. Applies delay to next scheduled run.                              |
| `RateLimitError`              | Triggers exponential back-off (1m → 2m → 5m → 10m → 1h → 3h → 12h → 24h) and **keeps scheduling on**. |
| `InvalidAccessTokenError`     | Token problems. Deactivates scenario, notifies user.                                    |
| `InvalidConfigurationError`   | Wrong configuration. Deactivates scenario, notifies user.                               |
| `DuplicateDataError`          | Reports as warning; doesn't interrupt.                                                  |
| `IncompleteDataError`         | Incoming data flagged incomplete.                                                       |
| `OutOfSpaceError`             | User storage exhausted.                                                                 |

> Older Make/Integromat docs reference `InconsistencyError`, `MaxResultsExceededError`, `UnknownError`, `UnexpectedError` — these **do not exist** in the current schema. Don't use them.

## Choosing the right type

| Symptom                             | Type                                |
| ----------------------------------- | ----------------------------------- |
| API returns 401/403                 | `InvalidAccessTokenError`           |
| API returns 429                     | `RateLimitError`                    |
| Network timeout / 5xx               | `ConnectionError`                   |
| 400/422 with field validation       | `DataError`                         |
| Duplicate detection ("already exists") | `DuplicateDataError`             |
| Generic unrecoverable               | `RuntimeError`                      |

## `response.error` schema

```json
{
  "response": {
    "error": {
      "type":    "RuntimeError",
      "message": "[{{statusCode}}] {{body.error.message}}",

      "400": { "type": "DataError",                 "message": "[{{statusCode}}] {{body.error.message}}" },
      "401": { "type": "InvalidAccessTokenError",   "message": "Authentication failed" },
      "403": { "type": "RuntimeError",              "message": "Forbidden: {{body.message}}" },
      "404": { "type": "DataError",                 "message": "Not found: {{body.message}}" },
      "429": { "type": "RateLimitError",            "message": "{{body.message}}" },
      "500": { "type": "ConnectionError",           "message": "Server error" }
    }
  }
}
```

Status keys can be numeric (`"404"`) or ranges (`"400-408"`). The non-status `message`/`type` is the fallback.

## `response.valid` — overriding HTTP-status judgment

When the API returns 200 OK but the body indicates failure.

### Simple
```json
"valid": "{{!body.error}}"
```

### Full with custom message + type
```json
"valid": {
  "condition": "{{body.status != 'error'}}",
  "message":   "{{body.error_description}}",
  "type":      "DataError"
}
```

### Combined (HTTP success + body error)
```json
{
  "response": {
    "valid": { "condition": "{{body.status != 'error'}}" },
    "error": {
      "200": { "message": "{{body.message}}", "type": "DataError" },
      "message": "[{{statusCode}}] {{body.reason}}"
    }
  }
}
```

### Empty result is failure (Get-style modules)
```json
{
  "response": {
    "valid": { "condition": "{{length(body.results) > 0}}", "message": "Not found" }
  }
}
```

## Error message resolution order

When `valid` fails:
1. `response.valid.message`
2. `response.error.<statusCode>.message`
3. `response.error.message`
4. Default: `"Response marked as invalid."`

(Same precedence for `type`.)

## Best practices

### Build user-friendly messages
```json
"message": "[{{statusCode}}] {{body.error.message}} (error code: {{body.error.code}})"
```

Include the status code, the API's human-readable message, and any error code.

### Always handle 429 properly
```json
"429": { "type": "RateLimitError", "message": "{{body.message}}" }
```

This triggers Make's auto-retry with exponential backoff. **Never leave 429 as `RuntimeError`** — it'll exhaust the 3-error counter and deactivate the user's scenario.

### Always handle 401/403 properly
```json
"401": { "type": "InvalidAccessTokenError", "message": "Token expired" }
```

Make notifies the user to reauthorize.

### `Retry-After` header (when API provides it)
```json
"repeat": {
  "condition": "{{statusCode == 429}}",
  "delay":     "{{headers.`retry-after` * 1000}}",
  "limit":     3
}
```

`repeat` retries the same request after a delay. Useful as a softer alternative to `RateLimitError`.

## Sanitize sensitive data in errors

Error messages should NOT leak credentials. If interpolating body that might contain a token:
```json
"message": "[{{statusCode}}] {{body.error.message}}"
```
Avoid `{{body}}` (full body) in messages — it may include sensitive fields.

## Source URLs

- https://developers.make.com/custom-apps-documentation/app-components/base/error-handling.md
- https://developers.make.com/custom-apps-documentation/component-blocks/api/handling-responses/error.md
- https://developers.make.com/custom-apps-documentation/best-practices/base/error-handling.md
- https://developers.make.com/custom-apps-documentation/best-practices/base/429-error-handling.md
