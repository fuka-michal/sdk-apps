# Schemas — Canonical Reference

Extracted from the VS Code "Make Apps Editor" extension's JSON Schemas at `~/.vscode/extensions/integromat.apps-sdk-*/syntaxes/imljson/schemas/`. This is the **source of truth** — when the docs and the schema disagree, the schema wins.

## File → Schema mapping

| File pattern                                                | Schema           |
| ----------------------------------------------------------- | ---------------- |
| `parameters.imljson`, `*.params.iml.json`                   | `parameters.json`|
| `expect.imljson`, `*.mappable-params.iml.json`              | `parameters.json`|
| `interface.imljson`, `*.interface.iml.json`                 | `parameters.json`|
| `common.imljson`, `common.json`                             | `common.json`    |
| `api.imljson`, `*.communication.iml.json`                   | `api.json`       |
| `attach.imljson`, `*.attach.iml.json`                       | `api.json`       |
| `detach.imljson`, `*.detach.iml.json`                       | `api.json`       |
| `publish.imljson`, `*.publish.iml.json`                     | `api.json`       |
| `samples.imljson`, `*.samples.iml.json`                     | `samples.json`   |
| `scopes.imljson`, `*.scope-list.iml.json`                   | `scopes.json`    |
| `scope.imljson`, `*.default-scope.iml.json`, `*.required-scope.iml.json` | `scope.json` |
| `epoch.imljson`, `*.epoch.iml.json`                         | `epoch.json`     |
| `base.imljson`, `base.iml.json`                             | `base.json`      |
| `api-oauth.imljson`, `*.oauth-communication.iml.json`       | `api-oauth.json` |
| `groups.json`                                               | `groups.json`    |
| `makecomapp.json`                                           | `makecomapp.schema.json` |

The `communication.imljson` file in a `connections/<name>/` folder for **OAuth** connections matches the `api-oauth.imljson` pattern (uses the OAuth schema with `preauthorize`/`authorize`/`token`/`info`/`refresh`/`invalidate`). All other communication files use the plain `api.json` schema (single request OR array of requests).

## Canonical enums

### Request `type` (request body encoding)
`json` (default) · `urlencoded` · `multipart/form-data` · `text` · `string` · `raw` · `binary`

### Response `type` (response parsing)
`automatic` (default) · `json` · `urlencoded` · `text` · `raw` · `string` · `binary` · `xml`

### Respond `type` (webhook response body encoding — only 3!)
`json` (default) · `urlencoded` · `text`

### HTTP `method` (only 5!)
`GET` (default) · `POST` · `PUT` · `PATCH` · `DELETE`

> `OPTIONS`, `HEAD` are NOT in the schema. If you need them, use IML in the method field.

### Error `type` — canonical 9 (no others exist)
- `RuntimeError` (default)
- `DataError`
- `RateLimitError`
- `OutOfSpaceError`
- `ConnectionError`
- `InvalidConfigurationError`
- `InvalidAccessTokenError`
- `IncompleteDataError`
- `DuplicateDataError`

> `InconsistencyError`, `MaxResultsExceededError`, `UnknownError`, `UnexpectedError` mentioned in old docs do NOT exist in the schema. Don't use them.

### Parameter `type` — canonical 31
`any` · `array` · `banner` · `boolean` · `buffer` · `cert` · `collection` · `color` · `date` · `email` · `file` · `filename` · `filter` · `folder` · `hidden` · `integer` · `json` · `number` · `password` · `path` · `pkey` · `port` · `select` · `text` · `time` · `timestamp` · `timezone` · `udt` · `uinteger` · `url` · `uuid`

> Special types I previously missed: **`banner`** (info banner displayed in module UI) and **`udt`** (Universal Data Type).

### OAuth 1 `signature_method`
`HMAC-SHA1` (default) · `RSA-SHA1` · `PLAINTEXT`

### OAuth 1 `transport_method`
`query` · `body` · `header` (default)

### AWS `sign_version`
`"2"` (default) · `"4"`

### Trigger `type`
`date` · `id`

### Trigger `order`
`asc` · `desc` · `unordered`

### Metadata `type` (connection display)
`text` · `email`

### Tags HTML handling (in `parameters.imljson`)
`strip` · `stripall` · `escape`

### Mode (for select/text with editable)
`edit` · `choose`

### Connection type (in `makecomapp.json` components.connection.*)
`basic` · `oauth`

> `oauth1`, `oauth2`, `jwt` are NOT distinct connectionType values. OAuth1 vs OAuth2 is determined by the phases used (`requestToken`/`accessToken` = OAuth1; `authorize`/`token` = OAuth2). JWT uses `basic` with the `jwt()` IML function.

### Webhook type
`web` (dedicated) · `web-shared` (shared)

### Module subtype (in `makecomapp.json` components.module.*.moduleType)
`action` · `instant_trigger` · `responder` · `search` · `trigger` · `universal`

### Action CRUD (`actionCrud` in makecomapp.json)
`create` · `read` · `update` · `delete`

## Validation rules

### Request body REQUIRED when method is one of:
- `api.json`: `POST`, `PUT`, `PATCH` → also requires `url`
- `base.json`: `POST`, `PUT`, `PATCH`, `DELETE` → also requires `url`

### Trigger response: required keys
- `type`, `order`, `id` always required
- `date` required when `type === "date"`

### Epoch response.output: required keys
- `date` AND `label` (always)

### Iterate (object form): required key
- `container`

### Pagination
All keys optional; `mergeWithParent` defaults to `true`.

### Parameter
`type` is the only required key.

### Options inner items
`value` always required; `label` is optional but recommended.

### Groups (app-level)
Each item requires `label` and `modules`.

### Connection metadata: required keys
- `value` (string up to 512 chars)
- `type` (`text` | `email`) — auto-detected if omitted

### Error spec
Top-level `message` is required if using object form. Status-code-keyed entries require `message`.

## Field type cheatsheet (parameter)

These are all the keys a parameter object can have (per schema):

| Key            | Type                              | Notes                                                  |
| -------------- | --------------------------------- | ------------------------------------------------------ |
| `name`         | string (≤128)                     | Required for input/output params (not for `rpc://...` ref) |
| `type`         | string (enum) or IML string       | **Required.** See parameter types enum above.          |
| `label`        | string (≤256)                     | UI label.                                              |
| `help`         | string                            | UI help text.                                          |
| `semantic`     | string                            | e.g. `"file:data"`, `"file:name"`.                     |
| `default`      | any                               | Default value.                                         |
| `advanced`     | boolean                           | Hides behind "Show advanced".                          |
| `required`     | boolean                           | Mark required.                                         |
| `grouped`      | boolean                           | Enables grouped options syntax.                        |
| `dynamic`      | boolean                           | —                                                      |
| `multiline`    | boolean                           | (text) Textarea instead of single-line.                |
| `sort`         | string                            | Items unsorted by default; this sorts them.            |
| `sequence`     | boolean                           | (collection) Preserve property order.                  |
| `schema`       | JSON Schema                       | —                                                      |
| `tags`         | `strip`/`stripall`/`escape`       | HTML handling.                                         |
| `coder`        | boolean                           | —                                                      |
| `multiple`     | boolean                           | (select) Allow multi-select.                           |
| `visible`      | string                            | IML visibility condition.                              |
| `convert`      | object                            | UI display conversion: `{ type, format, timezone, name, label, names }`. |
| `editable`     | bool OR `{enabled, help}`         | Map toggle.                                            |
| `mappable`     | bool OR `{help}`                  | Default true; map toggle.                              |
| `time`         | boolean                           | (date) `false` → date-only picker.                     |
| `rpc`          | `{url, label, parameters[]}`      | Search button.                                         |
| `mode`         | `edit`/`choose` or IML            | Initial mode for editable select.                      |
| `labels`       | object                            | Custom button labels.                                  |
| `metadata`     | object                            | Custom metadata.                                       |
| `spec`         | parameter OR parameters[]         | Sub-schema for `array` / `collection`.                 |
| `codepage`     | string                            | (buffer) Encoding.                                     |
| `pattern`      | string                            | Regex validation.                                      |
| `nested`       | string/array/`{store, domain}`    | Conditional fields.                                    |
| `options`      | array/string/object               | See options shapes below.                              |
| `extension`    | string or array of strings        | (file) Allowed extensions.                             |
| `validate`     | object                            | Type-specific: `{max, min, minItems, maxItems, pattern, enum}`. |
| `title`, `text`, `closable`, `theme`, `badge` | — | (banner) Banner type only.                |

### `options` shapes

1. **Inline array of options:**
   ```json
   [{ "label": "A", "value": "a" }, { "label": "B", "value": "b", "default": true }]
   ```

2. **Grouped options:**
   ```json
   [
     { "label": "Group 1", "options": [{"label":"A","value":"a"}] },
     { "label": "Group 2", "options": [{"label":"B","value":"b"}] }
   ]
   ```

3. **RPC URL string:**
   ```json
   "rpc://listFoo"
   ```

4. **Object with `store`** (RPC-based with extras):
   ```json
   {
     "store": "rpc://listFoo",      // or array of options/groups
     "label": "name",                // tells Make which key in RPC items = label
     "value": "id",                  // which key = value
     "default": "...",
     "learning": true,
     "placeholder": "Pick one",      // or { label, nested }
     "nested": [...],                // appears when option chosen
     "operators": [...],             // for filter type
     "logic": "and|or",              // for filter type
     "scope": [...],                 // OAuth scopes required for select
     "ids": true,                    // for folder picker
     "showRoot": true,
     "singleLevel": true
   }
   ```

### `nested` shapes

1. **RPC URL string:** `"rpc://dynamicFields"`
2. **Inline parameters array:** `[{ "name": "x", "type": "text" }]`
3. **Object form:**
   ```json
   {
     "store": "rpc://...",      // or inline parameters
     "domain": "some-domain"
   }
   ```

### Inner option fields

| Key           | Type    | Notes                                       |
| ------------- | ------- | ------------------------------------------- |
| `label`       | string  | UI label.                                   |
| `value`       | any     | **Required.** Value stored.                 |
| `description` | string  | Subtitle / detail.                          |
| `nested`      | nested  | Conditional fields when this option chosen. |
| `default`     | boolean | Mark as default.                            |
| `short`       | string  | Short label form.                           |

## Request directives — full list (api.json + sources.json)

```
url                    string (IML, ≤512), required for body-bearing requests
encodeUrl              boolean (default true)
method                 enum (GET/POST/PUT/PATCH/DELETE) or IML
headers                flat object {name: scalar | array}
qs                     flat object {name: scalar | array}
ca                     string (≤8192) - custom CA PEM
body                   any
type                   request-type enum or IML
temp                   object — set BEFORE request
condition              bool/string OR { condition, default }
gzip                   boolean (default false) — api only, not base
aws                    { key, secret, session?, bucket?, sign_version? }
followRedirects        boolean (default true)
followAllRedirects     boolean (default true)
log                    { sanitize: [string] }
oauth                  oauth-1 directive (rare)
pagination             { mergeWithParent?, url?, method?, headers?, qs?, body?, condition? }
output                 any (request-level: top-level emit; not common)
iterate                string OR { container, condition }   (request-level too)
respond                { type, status, headers, body }     (webhook only)
verification           { condition, respond }              (webhook only)
repeat                 { condition, delay, limit }
response               sub-object (see below)
```

## Response directives — full list (response.json)

```
type        response-type enum, status-keyed object, or IML
valid       bool/string OR { condition, message, type }
limit       number or string
error       string OR { message, type, "400"|"401"|...: {message, type} }
iterate     string OR { container, condition }
temp        object — set AFTER request
output      any
trigger     { type, order, id, date? } — REQUIRED [type, order, id]
data        object — persists into module data
metadata    { value, type }                    — connection display
uid         string or number                   — shared webhook routing
oauth       oauth-1 directive
wrapper     any (default {{output}})
expires     string (date)                      — connection refresh
```

> Note: `respond` / `verification` are **request-level** directives in webhook communication, NOT inside `response`.

## Base directives — full list (base.json)

```
url, baseUrl, encodeUrl, method, headers, qs, ca, body, type, temp,
condition, aws, followRedirects, followAllRedirects, log, oauth,
pagination, response

response: type, valid, limit, error, iterate, temp, output, data,
          metadata, uid, oauth, wrapper
```

> Base does NOT have: `gzip`, `respond`, `verification`, `repeat`, `output` (request-level), `iterate` (request-level), `trigger`, `expires`.

## OAuth communication file (api-oauth.json)

Top-level phases:
```
preauthorize   request
authorize      request (URL the user gets redirected to)
token          request (exchanges code → token)
info           request (validates + builds metadata/uid)
refresh        request (refreshes the token)
invalidate    request (revokes the token)
```

Each is a regular api.json request. `refresh` typically has `condition: "{{data.expires < addMinutes(now, 15)}}"`.

## Epoch (epoch.json)

Single api.json-shaped request with these differences:
- `response.output` is **required**.
- `response.output` must include `date` AND `label`.
- No `output` at request level.
- No `pagination`, `repeat`, `respond`, `verification` (just iterate + output).

## Groups (app-level)

Top-level array; each item:
```json
{ "label": "<group name>", "modules": ["moduleId1", "moduleId2"] }
```

File path is `groups.json` per the schema mapping (though `groups.imljson` works in practice — keep whatever your repo uses).

## Scope vs Scopes

- `scope.imljson` — array of scope strings the module/webhook/connection requires. Schema: `[string, ...]`.
- `scopes.imljson` — object mapping every scope key → human description. Schema: `{ "<scope>": "<description>" }`.

## Samples & Common

Both `samples.imljson` and `common.imljson` have schema `{ "type": "object" }` with no restrictions on inner keys. They're free-form.

## `makecomapp.json` structure

```json
{
  "fileVersion": 1,
  "generalCodeFiles": {
    "base":    "<path or null>",
    "common":  "<path or null>",
    "groups":  "<path or null>",
    "readme":  "<path or null>"
  },
  "components": {
    "connection": { "<id>": { codeFiles, label, connectionType, ... } },
    "webhook":    { "<id>": { codeFiles, label, webhookType, connection?, altConnection? } },
    "module":     { "<id>": { codeFiles, label, description, moduleType, actionCrud?, connection?, altConnection?, webhook? } },
    "rpc":        { "<id>": { codeFiles, label, connection? } },
    "function":   { "<id>": { codeFiles } }
  },
  "origins": [
    {
      "label":       "<friendly name>",
      "baseUrl":     "https://<zone>.make.com/api",
      "appId":       "<a-z, 0-9, dash; immutable>",
      "appVersion":  1,
      "apikeyFile":  "<relative or absolute path>",
      "idMapping":   { connection: [], function: [], module: [], rpc: [], webhook: [] }
    }
  ]
}
```

### `codeFiles` per component

```
attach           webhook only
code             function only (code.js)
common           connection only (common data file path)
communication    connection / module / rpc / webhook (api.imljson)
defaultScope     connection (scope.imljson)
detach           webhook only
epoch            module (trigger only)
installDirectives  connection (legacy install.imljson)
installSpec        connection (legacy install-spec.imljson)
interface        module / webhook (interface.imljson)
mappableParams   module (expect.imljson)
params           connection (parameters.imljson)
requiredScope    module / webhook (scope.imljson)
samples          module (samples.imljson)
scope            module (alternative to requiredScope)
scopeList        connection (scopes.imljson)
staticParams     module (parameters.imljson)
test             function only (test.js)
update           webhook only (update.imljson)
```

### Origin validation

- `baseUrl` pattern: `^https://(.*)/api$`
- `appId` pattern: `^[a-z][0-9a-z-]+[0-9a-z]$`
- `appVersion` is a number
- `apikeyFile` cannot contain the placeholder `- OR FILL`
- `label` cannot start with `-FILL-ME-`

## Notable differences from documentation

| Topic                      | Doc says                                 | Schema enforces                              |
| -------------------------- | ---------------------------------------- | -------------------------------------------- |
| Methods                    | Includes OPTIONS                         | Only GET/POST/PUT/PATCH/DELETE               |
| Error types                | InconsistencyError, UnknownError, etc.   | Only 9 listed above                          |
| Respond type               | json/urlencoded/multipart/text...        | Only json/urlencoded/text                    |
| Parameter types            | 26 types                                 | 31 types (incl. `banner`, `udt`)             |
| Connection type            | `oauth`/`oauth1`/`oauth2`/`jwt`/`basic`  | Only `basic` or `oauth` (in component metadata) |
| Webhook type               | `shared`, `dedicated`                    | `web` (dedicated) / `web-shared` (shared)    |
| Trigger required           | `type`, `order`, `id`, `date`            | `type`+`order`+`id`; `date` only if type=date|
