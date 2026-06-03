---
name: make-app
description: Build, edit, and debug Make.com (Integromat) Custom Apps (Apps SDK). Use when working with .imljson files, makecomapp.json, base / connections / modules / webhooks / rpc / functions folders, IML expressions ({{...}}), connection.accessToken, response.iterate / response.output / response.error, RPC URLs (rpc://...), OAuth2 / Basic / JWT connections, polling and instant triggers, dynamic options / fields / sample RPCs, pagination, log.sanitize, or any phrase like "Make app", "Integromat", "IML", "imljson".
---

# Make.com Custom Apps SDK

This skill helps build and edit Make Custom Apps. Make Apps are made of small `.imljson` files (JSON with `{{IML}}` templating) split across folders by component.

## Mental model

A Make app is a tree of **components**, each component is a folder of small JSON/JS files:

```
<app-root>/
  base.imljson                     Inherited HTTP defaults (baseUrl, headers, response, log)
  groups.imljson                   Module categorization in the scenario builder UI
  metadata.json                    App label/description/theme
  makecomapp.json                  (local dev only) Maps folder to remote Make app(s)
  connections/<name>/              Auth definitions (OAuth2 / Basic / JWT / OAuth1)
  modules/<name>/                  Action / Search / Trigger / Instant / Universal / Responder
  webhooks/<name>/                 Webhook receivers (shared or dedicated)
  rpc/<name>/                      Remote Procedure Calls (dynamic options / fields / sample)
  functions/<name>/                Custom IML JavaScript functions
```

### Who names each component

| Component   | Remote ID picked by                              | Local-folder convention                |
| ----------- | ------------------------------------------------ | -------------------------------------- |
| Connection  | **Make server** — `<appId><N>` (1, 2, 3…)        | Optional; usually matches the remote   |
| Webhook     | **Make server** — `<appId><N>` (shared counter)  | Optional; usually matches the remote   |
| Module      | **Developer** (mandatory, sent as-is)            | `camelCase` (e.g. `listRepositories`)  |
| RPC         | **Developer** (mandatory, sent as-is)            | `camelCase` (e.g. `getCustomFields`)   |
| Function    | **Developer** (mandatory; matches JS identifier) | `camelCase` (e.g. `removeEmpty`)       |

**Implication:** `attachedAccounts`, `connection`, `altConnection`, and instant-trigger `webhook` references use the **server-generated** name (`myapp-abc1`) — not the local folder name. The `idMapping` block inside `origins[]` in `makecomapp.json` records the local↔remote pairing. See `architecture.md` for the full regex table.

**Stable identity (`$id`).** When an app is synced to a **GitHub repo**, each connection/webhook `metadata.json` carries a `$id` (a UUID v4) that keeps its identity stable across folder/name renames and across cloning the repo into multiple apps. Make mints one on first push if absent, but **providing your own UUID v4 is recommended when hand-authoring** these files. See `architecture.md` → "Stable component identity".

Each `*.imljson` file is a section of a component. Common sections per folder:

| File                    | Purpose                                                                |
| ----------------------- | ---------------------------------------------------------------------- |
| `communication.imljson` | The HTTP request(s) + response handling (the "api" block)              |
| `parameters.imljson`    | Static config params (entered at install time; not mappable in scenes) |
| `expect.imljson`        | Mappable input params shown to the user inside a scenario              |
| `interface.imljson`     | Output schema (field names + types + labels) shown to mapping panel    |
| `samples.imljson`       | Example output bundle for preview                                      |
| `scope.imljson`         | Required OAuth scopes (array of strings)                               |
| `scopes.imljson`        | Map of all scopes to human descriptions (connection-level)             |
| `epoch.imljson`         | Polling-trigger "start from" picker config                             |
| `attach.imljson`        | Webhook registration call (dedicated webhooks)                         |
| `detach.imljson`        | Webhook unregistration call                                            |
| `update.imljson`        | Webhook update call when params change                                 |
| `metadata.json`         | Component metadata: `label`, `description`, `typeId`, `crud`, etc.     |

## Workflow

1. **Identify what's being built/changed.** Connection? Module? RPC? Webhook? Map intent to a folder.
2. **Pick the right module typeId** (see `modules.md`). The `metadata.json` `typeId` controls module behaviour.
3. **Inherit from base.imljson** — only set headers/auth in the connection's `info` (validation call) and in base. Module communication should not repeat auth headers if base already sets them.
4. **Sanitize logs.** Any auth token, password, secret, or OAuth code must appear in a `log.sanitize` list — otherwise logs are hidden entirely AND secrets leak.
5. **Validate** with IML-aware tooling (VS Code "Make Apps Editor" extension) before pushing.

## When to read which file

Read the relevant detail file with the Read tool when working on:

| Task                                                  | File to read                                |
| ----------------------------------------------------- | ------------------------------------------- |
| App folder layout, file naming, typeIds               | `architecture.md`                           |
| Setting up `base.imljson` (auth, error spec, log)     | `base.md`                                   |
| Building / fixing OAuth2 / Basic / JWT / OAuth1 conn  | `connections.md`                            |
| Action / Search / Trigger / Instant / Universal       | `modules.md`                                |
| Defining static / mappable / nested / file params     | `parameters.md`                             |
| Building the request: url, method, body, type, qs     | `communication.md`                          |
| Building the response: iterate, output, valid, temp   | `communication.md`                          |
| IML syntax, built-in variables, built-in functions    | `iml.md`                                    |
| Writing custom JavaScript helpers in `functions/`     | `iml-functions.md`                          |
| Dynamic dropdowns / dynamic fields / dynamic sample   | `rpc.md`                                    |
| Webhook attach/detach lifecycle, verification, uid    | `webhooks.md`                               |
| `response.error`, error types (RateLimitError, etc.)  | `errors.md`                                 |
| Paginating list endpoints                             | `pagination.md`                             |
| Naming, search-vs-action choice, polling trigger sort | `best-practices.md`                         |
| VS Code extension setup, Make CLI, makecomapp.json    | `tooling.md`                                |
| DevTool, Live Stream, debugging RPCs/IML/pagination   | `debugging.md`                              |
| App review, versioning, breaking changes              | `publishing.md`                             |
| Copy-paste starters (OAuth2 conn, Action, Search, …)  | `recipes.md`                                |
| Canonical JSON schemas (enums, required fields)       | `schemas-reference.md`                      |

**When the docs and the schema disagree, the schema wins.** `schemas-reference.md` is built from the actual JSON Schema files that ship with the VS Code extension — use it to settle ambiguity about enums, required fields, and validation rules.

## House rules

- **Never invent IML functions.** Only use those listed in `iml.md` or defined in `functions/`. If a transform is non-trivial, reach for `iml-functions.md`.
- **Indent imljson with tabs** (Make's editor saves with tabs). Keep keys in this order where applicable: `url`, `method`, `headers`, `qs`, `body`, `type`, `response`, `pagination`, `log`.
- **IML is sandboxed.** `==` is strict, arrays are 1-based with `-1` = last, backticks for header names with hyphens: `` headers.`X-API-Version` ``.
- **Polling triggers must sort descending.** See `best-practices.md` — using `asc`/`unordered` loses records past the 3200 cap.
- **Search modules must paginate + iterate**, Action modules must NOT.
- **Connections must validate** by calling an endpoint that fails on wrong credentials (`info` for OAuth, the main `url` for Basic).
- Never expose `accessToken`, `client_secret`, `code`, `password`, or `api_key` in unsanitized logs.

## Reference URL

- Docs root: https://developers.make.com/custom-apps-documentation
- Full LLM dump: https://developers.make.com/llms-full.txt
- Sitemap: https://developers.make.com/sitemap.md
