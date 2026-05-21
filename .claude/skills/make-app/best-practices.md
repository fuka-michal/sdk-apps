# Best Practices

Compact reference of the Make-approved patterns. Use when designing new components or refactoring.

## Naming

### App label
- Defer to brand guidelines.
- Title Case fallback; prepositions lowercase: `Instagram for Business`.
- Extra info in lowercase parentheses: `X (formerly Twitter)`.
- Never put descriptions in the name.

### Module internal name (folder name)
- Length 3-48 chars.
- Pattern `^[a-zA-Z][0-9a-zA-Z]+[0-9a-zA-Z]$`.
- Not a JS reserved word.
- Camel-case verb-noun: `listRepositories`, `createIssue`, `watchPullRequests`.

### Module label (UI display)
| Module type     | Pattern                    | Example                  |
| --------------- | -------------------------- | ------------------------ |
| Action (single) | `Verb [a/an] <singular>`   | `Create an issue`        |
| Search filtered | `Search <plural>`          | `Search repositories`    |
| List no-filter  | `List <plural>`            | `List repositories`      |
| Polling trigger | `Watch <plural>`           | `Watch new issues`       |
| Instant trigger | `Watch <plural> (instant)` | `Watch issues (instant)` |
| Universal       | `Make an API call`         | `Make an API call`       |

Sentence case (first word capitalized only). Suffix tags in lowercase parentheses: `(advanced)`, `(beta)`, `(deprecated)`.

### Module description
| Pattern                                           | Example                                                         |
| ------------------------------------------------- | --------------------------------------------------------------- |
| `Triggers when a/an <item> is <action>`           | `Triggers when a new issue is created`                          |
| `<Verbs> a/an <item> + details`                   | `Creates a new issue with title, body, and labels`              |
| `Returns a list of <items> + details`             | `Returns a list of repositories the user has access to`         |
| Universal verbatim                                | `Sends a custom API call to GitHub. You can use this to call endpoints that aren't covered by existing modules` |

### Parameter labels
- 1-3 words, sentence case.
- Match the third-party UI vocabulary, not API field names (`Owner`, not `owner_login`).
- Acronyms uppercase: `API`, `ID`, `URL`.
- No articles, no punctuation.
- Include `ID` in label if Map toggle is ON by default (mappable IDs).

### Output labels (`interface.imljson`)
- Sentence case, plain English (`Email address`, `User ID`).
- Mirror the input label of the corresponding mappable field.
- The **raw key** (the field name) stays as the API field name — only the label is humanized.

### Groups (`groups.imljson`)
- Apply once you have >10 modules.
- Group by entity (CONTACTS, ISSUES, REPOS) + OTHER bucket.
- Order: TRIGGERS first → generic groups → entity groups → OTHER.
- Within a group, RCUD order: Read → Create → Update → Delete.

### Connection labels
Include the auth type in lowercase parentheses: `MailerLite (API Key)`, `GitHub (OAuth)`.

## Module type choice

| Outcome                                    | Module type     |
| ------------------------------------------ | --------------- |
| One record in / 0 or 1 record out          | Action          |
| Many records out (filtered or unfiltered)  | Search          |
| Periodically pull new records              | Polling trigger |
| Push-based new records (webhook)           | Instant trigger |
| User-supplied custom endpoint              | Universal       |

Rules of thumb:
- **Action ≠ Search.** "List campaigns" with 100 records is a Search.
- **Action never paginates.** No `pagination` / no `iterate` / no `limit`.
- **Search always paginates** with iterate + limit, and always has a `limit` parameter (default 10, never required, never advanced).

## Polling trigger essentials

- Sort DESC on the server side. Always.
- `response.trigger` with `type` (`date` or `id`), `order: "desc"`, `id`, `date`.
- Store cursors via auto-managed `data.lastDate` / `data.lastID`.
- Limit `parameters.limit` default ≤ 100, capped at `min(300, 3 × perPage)`.
- Provide `epoch.imljson` for "start from" picker.

## Instant trigger essentials

- Label: `Watch <plural>` (no `(instant)` suffix unless app also has a polling Watch).
- Save authenticated identity into the connection (`response.uid` + `response.metadata`).
- Connection display name should reveal identity: `John Doe (john@example.com)`.
- For shared webhooks, the connection's `info` must set `uid`.

## Universal module requirements

Every approved Make app **must include** a universal "Make an API call" module:
- Label: `Make an API call`.
- Description (verbatim): `Sends a custom API call to <App>. You can use this to call endpoints that aren't covered by existing modules`.
- URL is relative to `base.baseUrl`.
- Pass through user-supplied URL, method, headers, qs, body.
- Output: `{ statusCode, headers, body }`.

## Connection essentials

- Every connection must validate by hitting an endpoint that fails on wrong credentials.
- For OAuth: place validation in `info`. For Basic/API key: in the `url` of `communication.imljson`.
- If no dedicated validation endpoint, hit `/me`, `/about`, or `/account`.
- Save identity in `response.metadata` (`{type: "email" | "text", value: ...}`).
- For shared webhooks, save `response.uid` = remote user ID.
- Mark `editable: true` only on non-URL fields (SSRF risk for subdomain fields).

## Base section best practices

- Static `baseUrl` preferred. If dynamic (per-user subdomain), use `{{connection.domain}}`.
- All module `url` values are **relative paths starting with `/`**.
- Repeated path segments (e.g. `/v2/`) belong in base, not modules.
- Set global error spec covering 401/403/404/429/500 in base.
- `log.sanitize` covers every secret used in headers/qs.

## Error handling

- 429 → `RateLimitError` (auto-backoff, keeps schedule on).
- 401/403 → `InvalidAccessTokenError`.
- Network/5xx → `ConnectionError`.
- Field validation 4xx → `DataError`.
- Build messages like `[{{statusCode}}] {{body.error.message}} (code: {{body.error.code}})`.

## Search modules — output shape

- Iterate over the records array.
- Output per-item: `"{{item}}"` (rest left to the interface).
- Always set `limit`.
- Never include `__IMTLENGTH__` / `__IMTINDEX__` in `interface.imljson`.

## Action modules — output shape

- Output `"{{body}}"` for simple create/update/get.
- If the API returns `{ data: {...} }`, output `"{{body.data}}"`.
- For delete: output `{ success: true, id: "{{parameters.id}}" }` (or whatever the API returns).

## Mapping toggle (`mode: "edit"`)

- Get/Update/Delete: `"mode": "edit"` (user can map raw IDs from previous modules).
- Search/List of high-cardinality entities: NO `mode: edit` (forces user to pick from RPC).
- Create: avoid `mode: edit` unless necessary (slows preloads).

## Date handling

- Always declare date fields as `"type": "date"` in `interface.imljson`.
- For input dates, use `"type": "date"` so Make handles formatting.
- Parse non-ISO dates in IML: `{{parseDate(item.timestamp, 'X')}}` (Unix).
- Output ISO 8601 strings via `formatDate`.

## File / Buffer handling

- Use the two-parameter pattern: `semantic: "file:name"` (text/filename) + `semantic: "file:data"` (buffer).
- Send via `type: "multipart/form-data"` body.
- Download modules return `{ name, data }`.

## Logging & sanitization

- Sanitize every header/qs/body field containing secrets.
- Without `log.sanitize`, Make hides the entire log from DevTool.
- Don't interpolate full `{{body}}` into error messages — leaks fields.

## Versioning

- For non-breaking changes, edit in place; Make tracks diffs and re-approves.
- For breaking API rewrites, create a NEW app (don't mutate the existing one).
- Mark obsolete parameters `[Deprecated]` and move to advanced; throw on use after grace period.

## Source URLs

- https://developers.make.com/custom-apps-documentation/best-practices/overview.md
- https://developers.make.com/custom-apps-documentation/best-practices/naming-conventions/apps.md
- https://developers.make.com/custom-apps-documentation/best-practices/naming-conventions/modules/modules-names.md
- https://developers.make.com/custom-apps-documentation/best-practices/naming-conventions/modules/module-labels.md
- https://developers.make.com/custom-apps-documentation/best-practices/naming-conventions/modules/module-descriptions.md
- https://developers.make.com/custom-apps-documentation/best-practices/naming-conventions/input-parameters/parameter-labels.md
- https://developers.make.com/custom-apps-documentation/best-practices/naming-conventions/output-labels.md
- https://developers.make.com/custom-apps-documentation/best-practices/naming-conventions/groups.md
- https://developers.make.com/custom-apps-documentation/best-practices/modules/module-types.md
- https://developers.make.com/custom-apps-documentation/best-practices/modules/search-modules.md
- https://developers.make.com/custom-apps-documentation/best-practices/modules/batch-actions.md
- https://developers.make.com/custom-apps-documentation/best-practices/modules/bulk-actions.md
- https://developers.make.com/custom-apps-documentation/best-practices/trigger-modules.md
- https://developers.make.com/custom-apps-documentation/best-practices/instant-triggers-scheduled.md
- https://developers.make.com/custom-apps-documentation/best-practices/base/base-url.md
- https://developers.make.com/custom-apps-documentation/best-practices/base/error-handling.md
- https://developers.make.com/custom-apps-documentation/best-practices/base/authorization-and-sanitization.md
- https://developers.make.com/custom-apps-documentation/best-practices/base/429-error-handling.md
- https://developers.make.com/custom-apps-documentation/best-practices/connections.md
- https://developers.make.com/custom-apps-documentation/best-practices/connections/editable-connection.md
- https://developers.make.com/custom-apps-documentation/best-practices/connections/connection-metadata.md
- https://developers.make.com/custom-apps-documentation/best-practices/connections/additional-oauth-scopes.md
- https://developers.make.com/custom-apps-documentation/best-practices/remote-procedure-calls.md
