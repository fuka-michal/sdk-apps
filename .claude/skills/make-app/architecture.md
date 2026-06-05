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

The repo folder name and the server-generated remote ID can differ. For freshly cloned apps the two are usually identical; they can diverge if you create the component in the repo first (with an arbitrary folder name) and then sync. The GitHub binding tracks this folder↔remote mapping via each connection/webhook's `$id` (see below).

## Stable component identity — the `$id` (UUID v4) in `metadata.json`

When an app is bound to a **GitHub repository** (Make's app ↔ GitHub sync), connection and webhook components carry a `$id` — a **UUID v4** stored inside their `metadata.json` — that acts as their *stable identity* across the repo↔Make boundary:

```json
{
  "$id": "550e8400-e29b-41d4-a716-446655440000",
  "label": "GitHub OAuth Connection",
  "type": "oauth"
}
```

**Why only connections and webhooks?** Their live names are **server-generated and mutable** (`<appId><N>`, see the table above), so neither the live name nor the repo folder name is a reliable identity. Modules, RPCs, and functions are developer-named and sent as-is — their folder name already *is* their stable identity, so they carry no `$id`.

**What the `$id` buys you:**

- **Rename-safe.** Rename the repo folder (`connections/oauth2-1/` → `connections/slack/`) and the next pull matches on the `$id`, updates the folder→name mapping, and leaves the live connection untouched. Without it, matching falls back to the folder name and a rename looks like a brand-new component.
- **Portable across apps.** The same repo can be cloned into many Make apps. Each app auto-generates its own local connection name, but every clone reuses the one `$id`, so the repo layout stays canonical and the mapping is never ambiguous.
- **Reference integrity.** Cross-component references (`attachedAccounts`, a module's `webhook`, `connectedSystemName`) are rewritten to the stable repo-folder path on push and back to the live name on pull, keyed off the `$id`.

**Why provide it yourself.** On the first push Make injects a random `$id` when one is absent, so it is not strictly mandatory. But if you **hand-author** a connection/webhook directly in the repo (or want deterministic, reviewable identity from commit one), set your own `$id` to any UUID v4 — a pull honors the `$id` already present in the file and only mints one when it is missing. Providing it up front avoids a churning "Make added `$id`" diff on the first sync and guarantees the component is recognized as the same entity everywhere the repo is cloned. The `$id` is a **repo-side-only** key: it lives in the GitHub `metadata.json` and is stripped before the body is written into the live app on pull.

The binding records the link as `$id → { dbName (live name), repoPath (repo folder) }`.

## Module `typeId` reference

The `metadata.json` `typeId` controls module behaviour. **Never change `typeId` on an existing module** — it changes its semantics.

| typeId | Module type      | Subtype name             | Purpose                                                          |
| ------ | ---------------- | ------------------------ | ---------------------------------------------------------------- |
| 1      | Polling trigger  | `trigger`                | Periodically fetches new items; uses `data.lastDate`/`lastID`    |
| 4      | Action           | `action`                 | Returns 0 or 1 result; no pagination / iterate                   |
| 9      | Search           | `search`                 | Returns multiple items; MUST have pagination + iterate           |
| 10     | Instant trigger  | `instant_trigger`        | Receives a single webhook bundle; pairs with `webhooks/<name>/`  |
| 11     | Responder        | `responder`              | Replies to inbound webhook request (rare)                        |
| 12     | Universal        | `universal`              | "Make an API call" — generic REST client                         |

`crud` hints at intent: `"create"`, `"read"`, `"update"`, `"delete"`. Used by Make for UI grouping; does not change behaviour.

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
