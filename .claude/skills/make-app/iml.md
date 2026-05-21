# IML — Integromat Markup Language

The templating syntax used inside every `.imljson` file. Mustache-like with `{{...}}` expressions.

## Syntax basics

```
{{expression}}                          single value
{{a + b}}                               arithmetic
{{'Hello, ' + item.name}}               string concat
{{body.data.users}}                     dot access
{{body.data[0]}}                        1-based array access (FIRST element)
{{body.data[-1]}}                       last element
{{headers.`X-API-Version`}}             backticks for keys with -, ., space
{{get(body, parameters.fieldName)}}     dynamic key
```

### IML quirks vs JavaScript
- `==` and `!=` are **strict** (no type coercion). No `===`/`!==`.
- **Arrays are 1-based.** `body.items[1]` = first item. `body.items[-1]` = last.
- Use backticks for property names containing `-`, `.`, or special chars: `` headers.`Content-Type` ``.
- No assignments, no loops, no function definitions inline. Only expressions.
- Sandboxed — no `require`, no `fs`, no `process`.

## Operators

| Category    | Operators                          |
| ----------- | ---------------------------------- |
| Algebraic   | `+` `-` `*` `/` `%`                |
| Equality    | `==` `!=` (strict)                 |
| Relational  | `<` `<=` `>` `>=`                  |
| Logical     | `&&` `\|\|` `!`                     |

## Built-in variables (context-dependent)

| Variable      | Available in           | Meaning                                                       |
| ------------- | ---------------------- | ------------------------------------------------------------- |
| `parameters`  | everywhere             | Module/RPC/webhook input values (mappable + static).          |
| `connection`  | everywhere (after auth)| Persisted connection data: `connection.accessToken`, etc.     |
| `common`      | everywhere             | Common Data (encrypted, app-wide): `common.clientId`, etc.    |
| `data`        | modules, attach/detach | Persistent module state: `data.lastDate`, `data.externalHookId`. |
| `body`        | inside `response.*`    | Response body of the current request.                         |
| `headers`     | inside `response.*`    | Response headers (case-insensitive).                          |
| `statusCode`  | inside `response.*`    | HTTP status of current response.                              |
| `temp`        | within module exec     | Cross-request scratchpad (`response.temp` sets it).           |
| `item`        | inside `iterate.output`| Current iteration element.                                    |
| `pagination`  | inside `pagination`    | `pagination.page` (1-based) auto-counter.                     |
| `iteration`   | rare                   | Loop index.                                                   |
| `now`         | everywhere             | Current UTC ISO 8601 timestamp.                               |
| `environment` | everywhere             | Environment / runtime context.                                |
| `query`       | webhook communication  | Inbound webhook URL query.                                    |
| `method`      | webhook communication  | Inbound webhook HTTP method.                                  |
| `payload`     | instant trigger        | Current webhook bundle.                                       |
| `webhook`     | attach/detach          | `webhook.url`, `webhook.<persisted-key>`.                     |
| `oauth`       | connection's `authorize` | `oauth.redirectUri`, `oauth.scope`.                          |
| `output`      | inside `wrapper`       | The transformed iteration array.                              |
| `scenario`    | rare                   | Scenario context.                                             |

## Built-in IML functions

### General
| Function                                | Returns                                             |
| --------------------------------------- | --------------------------------------------------- |
| `if(condition, then, else)`             | Ternary.                                            |
| `ifempty(value, fallback)`              | Fallback if value is null/undefined/empty string/array. |
| `switch(value, c1, r1, c2, r2, ..., default)` | First match wins.                             |
| `get(object, key)`                      | Dynamic property access.                            |
| `pick(object, keys...)`                 | Returns subset of object.                           |
| `omit(object, keys...)`                 | Returns object without listed keys.                 |
| `emptyarray`                            | Returns `[]`. (constant, no parens)                 |
| `emptyobject`                           | Returns `{}`.                                       |

### String
`ascii`, `base64`, `capitalize`, `contains`, `decodeURL`, `encodeURL`, `escapeHTML`, `escapeMarkdown`, `indexOf`, `length`, `lower`, `md5`, `mime`, `replace`, `replaceEmojiCharacters`, `sha1`, `sha256`, `sha512`, `split`, `startcase`, `stripHTML`, `substring`, `toBinary`, `toString`, `trim`, `upper`

Examples:
```
{{base64(connection.user + ':' + connection.pass)}}
{{md5('the quick brown fox')}}
{{sha256(parameters.payload)}}
{{lower(body.email)}}
{{trim(parameters.title)}}
{{replace(body.html, '<br>', '\n')}}
{{split(parameters.tags, ',')}}
{{substring(body.id, 0, 8)}}
{{stripHTML(body.html_description)}}
```

### Array
`add`, `contains`, `deduplicate` / `distinct`, `first`, `flatten`, `join`, `keys`, `last`, `length`, `map`, `merge`, `remove`, `reverse`, `shuffle`, `slice`, `sort`, `toArray`, `toCollection`

Examples:
```
{{join(parameters.tags, ', ')}}
{{distinct(body.ids)}}
{{merge(oauth.scope, parameters.extraScopes)}}
{{map(body.users, 'email')}}
{{length(body.items)}}
{{first(body.items)}}
{{last(body.items)}}
{{sort(body.items, 'created_at', 'desc')}}
{{slice(body.items, 0, 10)}}
{{contains(parameters.tags, 'urgent')}}
```

### Date/time
`formatDate(date, format, [tz])`, `parseDate(string, [format])`, `addDays`, `addHours`, `addMinutes`, `addMonths`, `addSeconds`, `addYears`, `setSecond`, `setMinute`, `setHour`, `setDay`, `setDate`, `setMonth`, `setYear`

Examples:
```
{{formatDate(now, 'YYYY-MM-DDTHH:mm:ssZ')}}
{{formatDate(item.created_at, 'YYYY-MM-DD', 'Europe/Prague')}}
{{parseDate(body.timestamp, 'X')}}                # X = Unix seconds
{{addSeconds(now, body.expires_in)}}
{{addMinutes(now, -5)}}                            # 5 minutes ago
```

Format tokens (Moment.js style): `YYYY`, `MM`, `DD`, `HH`, `mm`, `ss`, `Z`, `X` (Unix), `x` (Unix ms).

### Math
`abs`, `average`, `ceil`, `floor`, `formatNumber`, `max`, `median`, `min`, `parseNumber`, `round`, `stdevP`, `sum`, `trunc`

Examples:
```
{{round(parameters.price * 1.21, 2)}}
{{max(body.scores)}}
{{sum(body.amounts)}}
{{parseNumber('42')}}
{{formatNumber(body.value, 2, '.', ',')}}
```

### JSON
- `parseJSON(string)` → object/array
- `createJSON(object)` → string

```
{{parseJSON(body.payload)}}
{{createJSON(parameters.config)}}
```

### Crypto / encoding
- `jwt(payload, secret, algorithm, [options])` — algorithms: `HS256` (default), `HS384`, `HS512`, `RS256`. Options mirror `jsonwebtoken` npm package.

```
{{jwt({iss: connection.email, scope: 'foo', aud: 'bar', exp: addMinutes(now, 30), iat: now}, connection.privateKey, 'RS256')}}
```

## Common IML patterns

### Build authorization header
```
"authorization": "Bearer {{connection.accessToken}}"
"authorization": "Basic {{base64(connection.username + ':' + connection.password)}}"
"x-api-key": "{{connection.apiKey}}"
```

### Default value
```json
"per_page": "{{ifempty(parameters.limit, 30)}}"
```

### Conditional value
```json
"sort": "{{if(parameters.newest, 'desc', 'asc')}}"
```

### Build scope string (OAuth)
```json
"scope": "{{join(distinct(merge(oauth.scope, ifempty(parameters.scopes, emptyarray))), ' ')}}"
```

### Optional fields — drop nulls
Wrap body in a custom IML function that removes empty keys:
```json
"body": "{{removeEmpty(parameters)}}"
```
Or inline merge form:
```json
"body": { "fixed": "value", "{{...}}": "{{removeEmpty(parameters)}}" }
```

### Idempotent polling cursor
```json
"qs": { "since": "{{ifempty(data.lastDate, '1970-01-01T00:00:00Z')}}" }
```

### Parse timestamp formats
```
{{parseDate(item.created, 'YYYY-MM-DD HH:mm:ss')}}
{{parseDate(item.timestamp, 'X')}}                 # Unix seconds
{{parseDate(item.timestamp_ms, 'x')}}              # Unix ms
{{formatDate(item.created, 'YYYY-MM-DDTHH:mm:ssZ')}}
```

## Limits

- IML expressions run in a sandbox.
- No I/O, no module imports, no global state.
- For non-trivial transformations use `functions/<name>/code.js` — see `iml-functions.md`.

## Source URLs

- https://developers.make.com/custom-apps-documentation/block-elements/iml.md
- https://developers.make.com/custom-apps-documentation/block-elements/types.md
- https://developers.make.com/custom-apps-documentation/app-components/iml-functions.md
