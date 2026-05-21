# Parameters

Used in `parameters.imljson` (static), `expect.imljson` (mappable), `interface.imljson` (output), and RPC `parameters.imljson`. Same schema everywhere — just changes context.

## Universal parameter fields

| Field        | Type                              | Default | Meaning                                                          |
| ------------ | --------------------------------- | ------- | ---------------------------------------------------------------- |
| `name`       | string (≤128)                     | —       | Reference value via `{{parameters.name}}`.                       |
| `type`       | string (enum, ≤32) or IML         | —       | **Required.** One of the types below.                            |
| `label`      | string (≤256)                     | name    | Display label.                                                   |
| `help`       | string                            | —       | Help text shown below the field.                                 |
| `required`   | boolean                           | false   | Mark field as required.                                          |
| `default`    | any                               | —       | Default value.                                                   |
| `advanced`   | boolean                           | false   | Hidden behind "Show advanced" toggle.                            |
| `mappable`   | boolean OR `{help}`               | true    | (input) Show Map toggle. Object form provides alt help text.     |
| `editable`   | boolean OR `{enabled, help}`      | false   | (select) Allow manual entry. Object form provides alt help.      |
| `multiline`  | boolean                           | false   | (text) Multi-line textarea.                                      |
| `multiple`   | boolean                           | false   | (select) Allow multi-select.                                     |
| `grouped`    | boolean                           | false   | Enables grouped options syntax.                                  |
| `dynamic`    | boolean                           | —       | —                                                                |
| `sequence`   | boolean                           | —       | (collection) Preserve property order.                            |
| `sort`       | string                            | —       | Sort options instead of leaving unsorted.                        |
| `visible`    | string                            | —       | IML expression controlling visibility.                           |
| `time`       | boolean                           | true    | (date) `false` → date-only (no time component).                  |
| `tags`       | `strip`/`stripall`/`escape` or IML| —       | HTML handling in input.                                          |
| `codepage`   | string                            | —       | (buffer) Encoding.                                               |
| `extension`  | string OR string[]                | —       | (file) Allowed extensions.                                       |
| `pattern`    | string                            | —       | Regex validation (top-level convenience for validate.pattern).   |
| `convert`    | object                            | —       | UI display conversion: `{ type, format, timezone, name, label, names }`. |
| `options`    | array / string / object           | —       | See **options** below.                                           |
| `nested`     | string / array / `{store, domain}`| —       | Conditional fields revealed when this field has a value.         |
| `spec`       | parameter OR parameters[]         | —       | Sub-schema for `array` / `collection`.                           |
| `rpc`        | `{url, label, parameters}`        | —       | Search button.                                                   |
| `mode`       | `"edit"`/`"choose"` or IML        | —       | Initial mode when `editable: true`.                              |
| `validate`   | object OR boolean                 | —       | Type-aware. See **validate** below.                              |
| `metadata`   | object                            | —       | Custom metadata (e.g. `{ "expect": "json" }`).                   |
| `labels`     | object                            | —       | Custom button labels.                                            |
| `semantic`   | string                            | —       | `"file:data"` / `"file:name"` — auto-wires buffer plumbing.      |
| `schema`     | JSON Schema                       | —       | —                                                                |
| `coder`      | boolean                           | —       | —                                                                |

### Banner-type only fields
For `type: "banner"` (info panel rendered in module UI):

| Field      | Type    | Meaning                |
| ---------- | ------- | ---------------------- |
| `title`    | string  | Banner title.          |
| `text`     | string  | Banner body text.      |
| `closable` | boolean | Show close button.     |
| `theme`    | string  | `light` or `dark`.     |
| `badge`    | string  | Badge title.           |

## Parameter types (canonical 31)

### Text-family
- `text` — string input. `multiline: true` for textarea.
- `email` — email with validation.
- `url` — URL with validation.
- `password` — masked input. Used for secrets in connections.
- `hidden` — invisible input (use sparingly).
- `filename` — filename string.
- `uuid` — UUID string.

### Numeric
- `number` — float. Allows negatives & decimals.
- `integer` — signed integer.
- `uinteger` — unsigned integer (≥0).
- `port` — TCP port (0-65535).

### Date/time
- `date` — date (+optional time). Use `time: false` for date-only picker.
- `time` — time only.
- `timestamp` — Unix timestamp.
- `timezone` — IANA timezone string (e.g. `Europe/Prague`).

### Boolean
- `boolean` — checkbox.

### Selection
- `select` — single dropdown. Use `options` array OR `rpc://...`. `multiple: true` for multi-select.

### Composite
- `array` — list of values. Use `spec` for item type.
- `collection` — nested object with sub-fields. Use `spec` for field list.

### Structured
- `json` — arbitrary JSON input.
- `any` — any data (used in universal modules).
- `buffer` — binary data (file content).
- `file` — file upload.
- `folder` — folder picker.
- `udt` — Universal Data Type (advanced; data abstracted across modules).

### Display-only
- `banner` — info panel rendered in module UI. Uses `title`/`text`/`closable`/`theme`/`badge` instead of `name`/`label`.

### Other
- `color` — color picker, returns hex.
- `path` — filesystem path string.
- `filter` — Make's built-in filter builder (used in some search modules; `options.operators` + `options.logic`).
- `cert` — TLS certificate (PEM).
- `pkey` — Private key (PEM).

## Examples

### Simple text field
```json
{ "name": "title", "type": "text", "label": "Title", "required": true }
```

### Select with static options
```json
{
  "name": "state", "type": "select", "label": "State", "default": "open",
  "options": [
    { "label": "Open",   "value": "open" },
    { "label": "Closed", "value": "closed" },
    { "label": "All",    "value": "all" }
  ]
}
```

### Select powered by RPC
```json
{
  "name": "repo", "type": "select", "label": "Repository", "required": true,
  "options": "rpc://listRepositories"
}
```

### Cascading selects (RPC store + nested)
```json
{
  "name": "repo", "type": "select", "label": "Repository",
  "options": {
    "store": "rpc://listRepositories",
    "nested": [
      {
        "name": "branch", "type": "select", "label": "Branch",
        "options": "rpc://listBranches"
      }
    ]
  }
}
```

### Conditional field via `nested` on an option
```json
{
  "name": "auth_type", "type": "select", "label": "Auth type",
  "options": [
    { "label": "API Key", "value": "key", "nested": [
      { "name": "apiKey", "type": "password", "label": "API Key", "required": true }
    ]},
    { "label": "OAuth", "value": "oauth", "nested": [
      { "name": "token", "type": "password", "label": "OAuth token", "required": true }
    ]}
  ]
}
```

### Search button (when results are huge)
```json
{
  "name": "userId", "type": "text", "label": "User ID",
  "rpc": {
    "label": "Search users",
    "url": "rpc://searchUsers",
    "parameters": [
      { "name": "query", "type": "text", "label": "Search query" }
    ]
  }
}
```

### Array of strings
```json
{
  "name": "labels", "type": "array", "label": "Labels",
  "spec": { "type": "text", "label": "Label" }
}
```

### Array of collections
```json
{
  "name": "headers", "type": "array", "label": "Headers",
  "spec": [
    { "name": "key",   "type": "text", "label": "Key",   "required": true },
    { "name": "value", "type": "text", "label": "Value", "required": true }
  ]
}
```

### Collection (nested object)
```json
{
  "name": "user", "type": "collection", "label": "User",
  "spec": [
    { "name": "first_name", "type": "text", "label": "First name" },
    { "name": "last_name",  "type": "text", "label": "Last name" },
    { "name": "email",      "type": "email", "label": "Email" }
  ]
}
```

### File upload (use the two-semantic pattern)
```json
[
  { "name": "fileName", "type": "filename", "label": "File name", "semantic": "file:name", "required": true },
  { "name": "fileData", "type": "buffer", "label": "File data", "semantic": "file:data", "required": true }
]
```

Send to the API with multipart/form-data — see `communication.md`.

### Validation

For non-array types:
```json
{
  "name": "rating", "type": "uinteger", "label": "Rating", "required": true,
  "validate": { "min": 1, "max": 5 }
}

{
  "name": "slug", "type": "text", "label": "Slug",
  "validate": { "pattern": "^[a-z0-9-]+$" }
}

{
  "name": "title", "type": "text", "label": "Title",
  "validate": { "min": 1, "max": 200 }
}
```

For `array` type (different keys!):
```json
{
  "name": "tags", "type": "array", "label": "Tags",
  "spec": { "type": "text" },
  "validate": { "minItems": 1, "maxItems": 10 }
}

{
  "name": "rating", "type": "array", "label": "Ratings",
  "spec": { "type": "uinteger" },
  "validate": { "enum": [1, 2, 3, 4, 5] }
}
```

### Banner (info panel in UI)
```json
{
  "type":  "banner",
  "theme": "light",
  "title": "Heads up",
  "text":  "Owner repos can be filtered by `affiliation` only.",
  "badge": "Info",
  "closable": true
}
```

### Options object form — RPC with extras
```json
{
  "name": "ownerId", "type": "select", "label": "Owner",
  "options": {
    "store":       "rpc://listOwners",
    "label":       "name",       // tells Make which field in RPC items is the label
    "value":       "id",         // which field is the value
    "placeholder": "Pick an owner"
  }
}
```

### Grouped options
```json
{
  "name": "country", "type": "select", "label": "Country",
  "grouped": true,
  "options": [
    { "label": "Europe", "options": [
      { "label": "Czech Republic", "value": "CZ" },
      { "label": "Slovakia", "value": "SK" }
    ]},
    { "label": "Americas", "options": [
      { "label": "United States", "value": "US" }
    ]}
  ]
}
```

### Nested via `{store, domain}` (delegate to RPC tree)
```json
{
  "name": "folder", "type": "folder", "label": "Folder",
  "nested": { "store": "rpc://listChildren", "domain": "files" }
}
```

### Dynamic parameters (RPC returns the entire parameter array)
Insert RPC reference as a string in the parameter array:
```json
[
  { "name": "title", "type": "text", "label": "Title" },
  "rpc://dynamicCustomFields"
]
```

The RPC's communication returns a list of parameter objects (see `rpc.md`).

## Parameter label rules (UI quality)

- 1-3 words, sentence case (`Email address`, not `Email Address`).
- Match the third-party UI vocabulary, not API field names (`Owner`, not `owner_login`).
- Acronyms uppercase: `API`, `ID`, `URL`.
- No articles, no punctuation.
- If Map toggle ON by default → include `ID` in label. Otherwise omit.

## Source URLs

- https://developers.make.com/custom-apps-documentation/block-elements/parameters.md
- https://developers.make.com/custom-apps-documentation/component-blocks/parameters.md
- https://developers.make.com/custom-apps-documentation/component-blocks/mappable-parameters.md
- https://developers.make.com/custom-apps-documentation/best-practices/naming-conventions/input-parameters/parameter-labels.md
- https://developers.make.com/custom-apps-documentation/best-practices/naming-conventions/output-labels.md
