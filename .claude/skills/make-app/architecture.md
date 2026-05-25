# Architecture & File Layout

## Folder layout

```
<app-root>/
├── base.imljson              Shared HTTP config inherited by every module / RPC
├── groups.imljson            Module groups for the scenario builder UI
├── metadata.json             App label, description, theme color, language
├── install.imljson           (legacy) typically `{}`
├── install-spec.imljson      (legacy) typically `[]`
├── icon.png                  App icon
├── README.md
├── makecomapp.json           Local-dev manifest (origins[]) for VS Code extension
├── .secrets/apikey           Local Make API key (gitignored)
│
├── connections/
│   └── <connection-name>/
│       ├── metadata.json     { "label": "...", "type": "oauth"|"basic" }  (only two valid values)
│       ├── communication.imljson  Auth flow (authorize/token/info/refresh for oauth)
│       ├── parameters.imljson     User-entered fields (clientId, apiKey...)
│       ├── scope.imljson          Default scopes the connection requests
│       ├── scopes.imljson         Map of every known scope → description
│       ├── install.imljson        (legacy) typically `{}`
│       └── install-spec.imljson   (legacy) typically `[]`
│
├── modules/
│   └── <module-name>/
│       ├── metadata.json     { "label", "description", "typeId", "crud", "attachedAccounts" }
│       ├── communication.imljson  The HTTP request (or array of requests)
│       ├── parameters.imljson     Static module config (rarely used outside triggers)
│       ├── expect.imljson         Mappable input fields shown in scenarios
│       ├── interface.imljson      Output schema (field names + types + labels)
│       ├── samples.imljson        Sample output for preview
│       ├── scope.imljson          OAuth scopes this module needs
│       └── epoch.imljson          Polling-trigger "start from" picker (triggers only)
│
├── webhooks/
│   └── <webhook-name>/
│       ├── metadata.json     { "label", "type": "web"|"web-shared" }
│       ├── communication.imljson  Inbound request handler
│       ├── parameters.imljson     Webhook config the user fills in
│       ├── attach.imljson         (dedicated, attached) Register URL with service
│       ├── detach.imljson         (dedicated, attached) Unregister
│       ├── update.imljson         (optional) Update existing registration
│       ├── publish.imljson        (optional) Publish-time registration hook
│       └── scope.imljson          OAuth scopes the webhook needs
│
├── rpc/                      (sometimes "rpcs/")
│   └── <rpc-name>/
│       ├── metadata.json     { "label" }
│       ├── communication.imljson  The fetch + transform
│       └── parameters.imljson     RPC input params (passed via rpc://name?key=val)
│
└── functions/
    └── <function-name>/
        ├── code.js           function <name>(args) { ... return ...; }
        └── test/test.js      it("...", () => { assert.strictEqual(...) })
```

## Component IDs — who picks the name

Every component has a **remote ID** (used by Make's backend, by other components, and by scenarios) and a **local ID** (the folder name in the repo). For connections and webhooks these can differ; for modules / RPCs / functions they're always the same.

| Component   | Who picks the remote ID                              | Format / regex                                       | Example                |
| ----------- | ---------------------------------------------------- | ---------------------------------------------------- | ---------------------- |
| Connection  | **Server**: `<appId><N>` (incremental, immutable)    | `^[a-zA-Z][0-9a-zA-Z-]{1,33}[0-9a-zA-Z]$` (3-35)     | `mfu-github-odpjbh1`   |
| Webhook     | **Server**: `<appId><N>` (incremental, immutable)    | `^[a-zA-Z][0-9a-zA-Z-]{1,33}[0-9a-zA-Z]$` (3-35)     | `mfu-github-odpjbh2`   |
| Module      | **Developer** (mandatory, sent as-is to server)      | `^[a-zA-Z][0-9a-zA-Z]{2,63}$` (3-64, no dashes)      | `listRepositories`     |
| RPC         | **Developer** (mandatory, sent as-is to server)      | `^[a-zA-Z][0-9a-zA-Z]{2,63}$` (3-64, no dashes)      | `getCustomFields`      |
| Function    | **Developer** (mandatory; must match JS identifier)  | `^[a-zA-Z][0-9a-zA-Z]{1,94}[0-9a-zA-Z]$` (3-96)      | `removeEmpty`          |
| **App ID**  | Developer (set at app creation, **immutable**)       | `^[a-z][0-9a-z-]+[0-9a-z]$` (lowercase + dash only)  | `mfu-github-odpjbh`    |

**Key implication:** When wiring `attachedAccounts`, `connection`, `altConnection`, or the instant-trigger `webhook` field, you must use the **server-generated remote name** (`mfu-github-odpjbh1`) — never the local folder name.

**`origins[].idMapping`** in `makecomapp.json` maps local folder names to remote IDs. For freshly cloned apps the two are usually identical; they can diverge if you create the local component first (with an arbitrary local label) and then deploy.

## Module `typeId` reference

The `metadata.json` `typeId` controls module behaviour. **Never change `typeId` on an existing module** — it changes its semantics.

| typeId | Module type      | makecomapp `moduleType`  | Purpose                                                          |
| ------ | ---------------- | ------------------------ | ---------------------------------------------------------------- |
| 1      | Polling trigger  | `trigger`                | Periodically fetches new items; uses `data.lastDate`/`lastID`    |
| 4      | Action           | `action`                 | Returns 0 or 1 result; no pagination / iterate                   |
| 9      | Search           | `search`                 | Returns multiple items; MUST have pagination + iterate           |
| 10     | Instant trigger  | `instant_trigger`        | Receives a single webhook bundle; pairs with `webhooks/<name>/`  |
| 11     | Responder        | `responder`              | Replies to inbound webhook request (rare)                        |
| 12     | Universal        | `universal`              | "Make an API call" — generic REST client                         |

`crud` (or `actionCrud` in `makecomapp.json`) hints at intent: `"create"`, `"read"`, `"update"`, `"delete"`. Used by Make for UI grouping; does not change behaviour.

## App-root `metadata.json`

```json
{
  "label": "GitHub App",
  "description": "Michals Github App",
  "language": "en",
  "theme": "#777a82",
  "global": true
}
```

| Field        | Meaning                                                |
| ------------ | ------------------------------------------------------ |
| `label`      | Display name in scenario builder                       |
| `description`| Tooltip / overview                                     |
| `language`   | ISO 639-1 (`en`, `cs`, ...)                            |
| `theme`      | Brand hex color (used in module tile borders / icons)  |
| `global`     | true = enabled across all teams (private apps)         |

## `makecomapp.json` (local development)

Generated by the VS Code "Make Apps Editor" when cloning. Multi-environment is supported via `origins[]`:

```json
{
  "origins": [
    {
      "label": "Production",
      "baseUrl": "https://eu1.make.com/api",
      "appId": "mfu-github-odpjbh",
      "appVersion": 1,
      "apikeyFile": "../.secrets/apikey"
    },
    {
      "label": "Testing",
      "baseUrl": "https://eu1.make.com/api",
      "appId": "mfu-github-odpjbh-test",
      "appVersion": 1,
      "apikeyFile": "../.secrets/apikey"
    }
  ]
}
```

The `appId` must match the regex `^[a-z][0-9a-z-]+[0-9a-z]$` and is **immutable after app creation**.

Other developers must add their own origin record; **never edit existing records** (it breaks their connection).

## `base.imljson` (legacy filename; same purpose as the Base component)

See `base.md`.

## `groups.imljson`

Array categorizing modules in the scenario picker. If only `"Other"` is present, modules auto-group by type.

```json
[
  { "label": "Repositories", "modules": ["listRepositories", "getRepository"] },
  { "label": "Issues",       "modules": ["listIssues", "createIssue", "updateIssue"] },
  { "label": "Other",        "modules": ["makeAPICall"] }
]
```

Every module must appear in at least one group. A module can appear in multiple groups.
