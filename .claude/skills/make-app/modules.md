# Modules

A module is one operation visible in the scenario builder. Six types — pick by `typeId` in `metadata.json`. **`typeId` is immutable after the module exists.**

## Naming — module IDs are user-picked and persistent

Unlike connections/webhooks (which get a server-generated `<appId><N>` name), **module IDs are entered by the developer and sent to Make as-is** — the same string is used for the local folder, the remote ID, and every cross-reference (groups, `attachedAccounts`, RPC parents, etc.).

| Aspect           | Detail                                                                |
| ---------------- | --------------------------------------------------------------------- |
| Picked by        | **Mandatory user input** — no autogenerate option                     |
| Convention       | `camelCase`, verb + noun (`listRepositories`, `createIssue`)          |
| Format regex     | `^[a-zA-Z][0-9a-zA-Z]{2,63}$` — 3-64 chars, alphanumeric only, letter start (**no dashes**) |
| Immutability     | Treat as immutable after deploy — renaming breaks groups, references, and scenarios already using it |
| Used in          | `groups.imljson` `modules[]`, `attachedAccounts` references, deep links |

Module ID rules are stricter than connection/webhook IDs — **no dashes allowed**, longer maximum (64 chars).

| typeId | Type            | When to use                                                          |
| ------ | --------------- | -------------------------------------------------------------------- |
| 1      | Polling trigger | Periodically fetch new items via REST (`Watch <plural>`)             |
| 4      | Action          | Single-record create / update / delete / get (returns 0 or 1 result) |
| 9      | Search          | List / search returning N records (must paginate)                    |
| 10     | Instant trigger | Receives a webhook; pairs with `webhooks/<name>/`                    |
| 11     | Responder       | Replies to inbound webhook request (very rare)                       |
| 12     | Universal       | "Make an API call" — user-supplied URL/method/body                   |

## Folder structure (per module)

```
modules/<name>/
├── metadata.json         label, description, typeId, crud, attachedAccounts
├── communication.imljson HTTP request + response handling
├── parameters.imljson    Static config (rarely used outside triggers)
├── expect.imljson        Mappable inputs the user fills in
├── interface.imljson     Output schema (mapping panel)
├── samples.imljson       Sample output bundle
├── scope.imljson         OAuth scopes this module requires
└── epoch.imljson         Polling-trigger "start from" picker only
```

`metadata.json` example:
```json
{
  "label": "List Repositories",
  "description": "Returns a list of repositories for the authenticated user.",
  "typeId": 9,
  "crud": "read",
  "attachedAccounts": ["mfu-github-odpjbh"]
}
```

`attachedAccounts` lists the connection names the module can use. Empty `[]` = no connection (rare).

## Action (typeId 4)

Returns at most one bundle. **No `iterate`, no `pagination`.**

```json
// modules/createIssue/communication.imljson
{
  "url": "/repos/{{parameters.owner}}/{{parameters.repo}}/issues",
  "method": "POST",
  "body": {
    "title": "{{parameters.title}}",
    "body": "{{parameters.body}}",
    "labels": "{{parameters.labels}}"
  },
  "response": { "output": "{{body}}" }
}
```

`expect.imljson`:
```json
[
  { "name": "owner", "type": "text", "label": "Owner", "required": true },
  { "name": "repo",  "type": "text", "label": "Repository", "required": true },
  { "name": "title", "type": "text", "label": "Title", "required": true },
  { "name": "body",  "type": "text", "label": "Body", "multiline": true },
  { "name": "labels","type": "array", "label": "Labels", "spec": { "type": "text" } }
]
```

Label format: `Verb a noun` ("Create an issue", "Get a repository", "Update a user").

## Search (typeId 9)

Returns N bundles. **Must have `pagination` + `iterate` + `response.limit`.** Use `parameters.imljson` limit field, default 10, never required, never advanced.

```json
// modules/listRepositories/communication.imljson
{
  "url": "/user/repos",
  "method": "GET",
  "qs": {
    "visibility":  "{{parameters.visibility}}",
    "affiliation": "{{parameters.affiliation}}",
    "sort":        "{{parameters.sort}}",
    "direction":   "{{parameters.direction}}",
    "per_page":    "{{ifempty(parameters.limit, 30)}}"
  },
  "pagination": {
    "qs": { "page": "{{pagination.page}}" }
  },
  "response": {
    "iterate": "{{body}}",
    "output":  "{{item}}",
    "limit":   "{{parameters.limit}}"
  }
}
```

Label format: `List <plural>` (no filters) or `Search <plural>` (with filters).

## Polling Trigger (typeId 1)

Fires for each new item seen since last run. Must include `response.trigger`, sort DESC server-side, and offer an `epoch.imljson` "start from" picker.

```json
// modules/watchIssues/communication.imljson
{
  "url": "/repos/{{parameters.owner}}/{{parameters.repo}}/issues",
  "method": "GET",
  "qs": {
    "state": "all",
    "sort": "created",
    "direction": "desc",
    "since": "{{data.lastDate}}",
    "per_page": "{{ifempty(parameters.limit, 100)}}"
  },
  "pagination": { "qs": { "page": "{{pagination.page}}" } },
  "response": {
    "iterate": "{{body}}",
    "trigger": {
      "type": "date",
      "order": "desc",
      "id": "{{item.id}}",
      "date": "{{item.created_at}}"
    },
    "output": "{{item}}",
    "limit": "{{parameters.limit}}"
  }
}
```

`response.trigger` fields (per schema):
| Field   | Values                              | Required           | Meaning                                          |
| ------- | ----------------------------------- | ------------------ | ------------------------------------------------ |
| `type`  | `"date"` or `"id"` (or IML)         | yes                | Cursor strategy                                  |
| `order` | `"asc"`, `"desc"`, `"unordered"`    | yes                | How the API returns items. Always `"desc"`.      |
| `id`    | IML string (≤512)                   | yes                | The current item's unique ID                     |
| `date`  | IML string (≤512)                   | yes if `type=date` | Current item's create/update timestamp           |

Why DESC: Make stops paginating once it sees a known item. With `asc`/`unordered`, items past the 3200 cap are lost.

State variables: `data.lastDate`, `data.lastID` — auto-persisted between runs.

### `epoch.imljson` — "start from" picker

`response.output` is **required** and must include `date` AND `label`. No `pagination`/`repeat`/`respond` here.

```json
{
  "url": "/repos/{{parameters.owner}}/{{parameters.repo}}/issues",
  "qs": { "state": "all", "sort": "created", "direction": "desc", "per_page": 100 },
  "response": {
    "iterate": "{{body}}",
    "output": {
      "id": "{{item.id}}",
      "date": "{{item.created_at}}",
      "label": "{{item.title}}"
    },
    "limit": 300
  }
}
```

User UI options: All / Since specific date / From now on / Choose manually.

### `parameters.imljson` for triggers (static fields)
```json
[
  { "name": "limit", "type": "uinteger", "label": "Limit", "default": 10, "required": true }
]
```

Triggers split UI fields into `parameters.imljson` (static, set at trigger creation) and `expect.imljson` (mappable per run). For most polling triggers, all fields go in `parameters.imljson`.

Label format: `Watch <plural>` ("Watch issues", "Watch new contacts", "Watch updated contacts").

## Instant Trigger (typeId 10)

Receives one webhook → emits one bundle. Pairs with a `webhooks/<name>/` folder.

```json
// modules/watchIssueInstant/metadata.json
{
  "label": "Watch issues (instant)",
  "description": "Triggers when a GitHub issue event occurs.",
  "typeId": 10,
  "crud": "read",
  "webhook": "issueWebhook"
}
```

`communication.imljson` is **optional** (used only if you need to fetch extra data per bundle). No `iterate` / `pagination` here.

See `webhooks.md` for the matching webhook setup.

## Universal Module (typeId 12)

Generic "Make an API call". Lets users hit endpoints not natively wrapped.

```json
// modules/makeAPICall/communication.imljson
{
  "url": "{{parameters.url}}",
  "method": "{{parameters.method}}",
  "qs": "{{parameters.qs}}",
  "headers": "{{parameters.headers}}",
  "body": "{{parameters.body}}",
  "response": {
    "output": {
      "statusCode": "{{statusCode}}",
      "headers": "{{headers}}",
      "body": "{{body}}"
    }
  }
}
```

`expect.imljson`:
```json
[
  { "name": "url", "type": "text", "label": "URL", "required": true, "help": "Relative path, e.g. /repos/owner/repo/issues" },
  { "name": "method", "type": "select", "label": "Method", "required": true, "default": "GET",
    "options": [
      { "label": "GET", "value": "GET" }, { "label": "POST", "value": "POST" },
      { "label": "PUT", "value": "PUT" }, { "label": "PATCH", "value": "PATCH" },
      { "label": "DELETE", "value": "DELETE" }
    ]
  },
  { "name": "qs", "type": "array", "label": "Query String", "spec": [
    { "name": "key", "type": "text" }, { "name": "value", "type": "text" }
  ] },
  { "name": "headers", "type": "array", "label": "Headers", "spec": [
    { "name": "key", "type": "text" }, { "name": "value", "type": "text" }
  ] },
  { "name": "body", "type": "any", "label": "Body" }
]
```

Approved Make apps **must include a universal module**. Description (verbatim): `Sends a custom API call to <App>. You can use this to call endpoints that aren't covered by existing modules`.

## Module label & description rules

| Type            | Label pattern              | Example                     | Description pattern                                |
| --------------- | -------------------------- | --------------------------- | -------------------------------------------------- |
| Action (single) | `Verb [a/an] <singular>`   | "Create an issue"           | `Creates a/an <entity> + details.`                 |
| Search (filter) | `Search <plural>`          | "Search repositories"       | `Returns a list of <items> + filter description.`  |
| List (no filt.) | `List <plural>`            | "List repositories"         | `Returns a list of <items>.`                       |
| Polling trigger | `Watch <plural>`           | "Watch issues"              | `Triggers when a/an <item> is <action>.`           |
| Instant trigger | `Watch <plural> (instant)` | "Watch issues (instant)"    | `Triggers immediately when …`                      |
| Universal       | `Make an API call`         | "Make an API call"          | Verbatim text above.                               |

Sentence case (first word only). Tags in lowercase parentheses: `(advanced)`, `(beta)`, `(deprecated)`.

## `expect.imljson` vs `parameters.imljson`

- `parameters.imljson` = static — set when the user *creates* the module. Used mostly by triggers (limit, filter criteria) and to gate behaviour at scenario-build time.
- `expect.imljson` = mappable — set per execution. Most action / search inputs go here.

Both files have the same parameter schema; see `parameters.md`.

## `interface.imljson` — output schema

Defines the field names + types + labels visible in the mapping panel of downstream modules.

```json
[
  { "name": "id", "type": "uinteger", "label": "ID" },
  { "name": "title", "type": "text", "label": "Title" },
  { "name": "state", "type": "text", "label": "State" },
  { "name": "created_at", "type": "date", "label": "Created at" },
  { "name": "user", "type": "collection", "label": "User", "spec": [
    { "name": "login", "type": "text", "label": "Login" },
    { "name": "id", "type": "uinteger", "label": "ID" }
  ]},
  { "name": "labels", "type": "array", "label": "Labels", "spec": {
    "type": "collection",
    "spec": [
      { "name": "name", "type": "text", "label": "Name" },
      { "name": "color", "type": "color", "label": "Color" }
    ]
  }}
]
```

Match `interface.imljson` field names + types to what `response.output` actually emits. Outputs not in the interface still appear in mapping, but unstyled.

Special interface keys to **strip** in search modules: `__IMTLENGTH__`, `__IMTINDEX__` (Make adds them automatically; don't include in interface).

## `samples.imljson`

Single bundle example used in the mapping panel preview. Match the shape of `interface.imljson`.

```json
{
  "id": 1234567,
  "title": "Example issue",
  "state": "open",
  "created_at": "2024-01-15T10:00:00Z",
  "user": { "login": "octocat", "id": 1 },
  "labels": [ { "name": "bug", "color": "ee0701" } ]
}
```

## `scope.imljson`

Per-module list of OAuth scopes the module needs. Empty `[]` for modules that don't need additional scopes.

```json
["repo", "write:issue"]
```

Adding a new scope triggers Make's "Authorize additional permissions" dialog for users when they install the new version.

## Source URLs

- https://developers.make.com/custom-apps-documentation/app-components/modules.md
- https://developers.make.com/custom-apps-documentation/app-components/modules/action.md
- https://developers.make.com/custom-apps-documentation/app-components/modules/search.md
- https://developers.make.com/custom-apps-documentation/app-components/modules/trigger.md
- https://developers.make.com/custom-apps-documentation/app-components/modules/instant-trigger.md
- https://developers.make.com/custom-apps-documentation/app-components/modules/universal-module.md
- https://developers.make.com/custom-apps-documentation/app-components/modules/responder.md
- https://developers.make.com/custom-apps-documentation/component-blocks/epoch.md
- https://developers.make.com/custom-apps-documentation/component-blocks/interface.md
- https://developers.make.com/custom-apps-documentation/component-blocks/samples.md
- https://developers.make.com/custom-apps-documentation/component-blocks/scope.md
- https://developers.make.com/custom-apps-documentation/best-practices/modules/module-types.md
- https://developers.make.com/custom-apps-documentation/best-practices/modules/search-modules.md
- https://developers.make.com/custom-apps-documentation/best-practices/trigger-modules.md
- https://developers.make.com/custom-apps-documentation/best-practices/instant-triggers-scheduled.md
