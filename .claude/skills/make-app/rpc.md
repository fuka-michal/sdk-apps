# Remote Procedure Calls (RPC)

RPCs fetch live data while the user is configuring a module in the scenario editor (e.g., populate a dropdown of repositories, project boards, custom fields).

Each RPC lives in `rpc/<name>/` (sometimes `rpcs/<name>/`).

## Folder structure

```
rpc/<name>/
├── metadata.json         { "label": "List Repositories" }
├── communication.imljson HTTP request + response handler
└── parameters.imljson    Input params (passed via rpc://name?key=value)
```

## Three RPC types

| Type            | Purpose                                                            | Output shape                            |
| --------------- | ------------------------------------------------------------------ | --------------------------------------- |
| Dynamic Options | Populate a `select` dropdown or Search button                      | `{ "label": "...", "value": "..." }`    |
| Dynamic Fields  | Generate the parameter list of a module at runtime                 | Parameter objects (like `expect.imljson`) |
| Dynamic Sample  | Provide a representative sample bundle (one record from the API)   | A single object matching the interface  |

The RPC type is determined by where it's referenced and what it returns — Make doesn't tag RPCs by type explicitly.

## Reference syntax

```
rpc://nameOfRpc                       Latest version
rpc://nameOfRpc@2                     Pinned to version 2
rpc://nameOfRpc?key=value&key2=val2   With params (accessible as parameters.key)
```

## Dynamic Options RPC

Populates a `select` field with live values.

`rpc/listRepositories/communication.imljson`:
```json
{
  "url": "/user/repos",
  "method": "GET",
  "qs": { "per_page": 100 },
  "pagination": { "qs": { "page": "{{pagination.page}}" } },
  "response": {
    "iterate": "{{body}}",
    "output": {
      "label": "{{item.full_name}}",
      "value": "{{item.name}}"
    },
    "limit": 300
  }
}
```

Use in a module's `expect.imljson`:
```json
{
  "name": "repo", "type": "select", "label": "Repository",
  "options": "rpc://listRepositories",
  "required": true
}
```

### Hardcoded (request-less) options
```json
{
  "response": {
    "output": [
      { "label": "Read",  "value": "read" },
      { "label": "Write", "value": "write" }
    ]
  }
}
```

### Filtered options
```json
{
  "response": {
    "iterate": {
      "container": "{{body.data}}",
      "condition": "{{item.active == true}}"
    },
    "output": { "label": "{{item.name}}", "value": "{{item.id}}" }
  }
}
```

### Cascading selects — `store` + `nested`
```json
{
  "name": "repo", "type": "select", "label": "Repository",
  "options": {
    "store": "rpc://listRepositories",
    "nested": [
      {
        "name": "branch", "type": "select", "label": "Branch",
        "options": "rpc://listBranches"
      },
      {
        "name": "label", "type": "select", "label": "Label",
        "options": "rpc://listLabels"
      }
    ]
  }
}
```

Inside `listBranches` the parent's selection is available as `parameters.repo` (the field's `name`).

### Search button (huge data sets)
```json
{
  "name": "userId", "type": "text", "label": "User ID",
  "rpc": {
    "label": "Search users",
    "url":   "rpc://searchUsers",
    "parameters": [
      { "name": "query", "type": "text", "label": "Search query" }
    ]
  }
}
```

## Dynamic Fields RPC

Returns parameter objects — Make builds the UI from them.

`rpc/getCustomFields/communication.imljson`:
```json
{
  "url": "/repos/{{parameters.owner}}/{{parameters.repo}}/labels",
  "method": "GET",
  "response": {
    "iterate": "{{body}}",
    "output": {
      "name":  "{{item.name}}",
      "label": "{{item.name}}",
      "type":  "text"
    }
  }
}
```

Used in `expect.imljson` by appending a `rpc://` string:
```json
[
  { "name": "owner", "type": "text", "label": "Owner", "required": true },
  { "name": "repo",  "type": "text", "label": "Repository", "required": true },
  "rpc://getCustomFields"
]
```

Or nested under a static select option:
```json
{
  "name": "useCustom", "type": "select", "label": "Mode",
  "options": [
    { "label": "Default", "value": "default" },
    { "label": "Custom",  "value": "custom", "nested": "rpc://getCustomFields" }
  ]
}
```

> Important: Make parameter types differ from raw API types. Use a custom IML function (`dynamicFields` — see `iml-functions.md`) to map service types → Make types.

## Dynamic Sample RPC

Returns one example record to populate sample data in the mapping panel.

```json
{
  "url": "/issues",
  "method": "GET",
  "qs":  { "per_page": 1 },
  "response": {
    "iterate": "{{body}}",
    "output":  "{{item}}",
    "limit":   1
  }
}
```

## RPC parameter passing

URL form:
```
rpc://searchUsers?query=octocat&active=true
```

Read inside the RPC's communication:
```json
{ "qs": { "q": "{{parameters.query}}", "active": "{{parameters.active}}" } }
```

Or define formal RPC params in `rpc/<name>/parameters.imljson`:
```json
[
  { "name": "query", "type": "text", "label": "Query" },
  { "name": "type",  "type": "select", "options": [ {"label":"User","value":"u"} ] }
]
```

## Available IML vars in RPC

`now`, `parameters`, `connection`, `common`, `data`, `body`, `headers`, `temp`. No `webhook`, no `payload`, no `data.lastDate`.

## Limits

| Limit                       | Value                                  |
| --------------------------- | -------------------------------------- |
| Execution timeout           | 40 seconds                             |
| Max API calls per RPC       | ~3 recommended                         |
| Total items recommended     | ≤ 3× page size (e.g. 75 when per_page=25), or up to 300-500 with large pages |
| `limit`                     | Always set on iterate                  |
| Pagination condition        | e.g. `"condition": "{{pagination.page <= 3}}"` |

## Best practices

- For Get / Update / Delete modules, set `"mode": "edit"` on the field so users can also map a raw ID.
- For Search / List modules with high-cardinality entities (customers, deals), **do not** set `mode: edit`.
- For Create modules, avoid `mode: edit` unless necessary — it triggers slow preloads.
- In mapping mode the user enters raw IDs (not labels), so RPC labels need to make IDs guessable.
- Provide a `limit` parameter in the *module* (not the RPC) so users can cap results when there are many.

## Source URLs

- https://developers.make.com/custom-apps-documentation/app-components/rpcs.md
- https://developers.make.com/custom-apps-documentation/app-components/rpcs/dynamic-options-rpc.md
- https://developers.make.com/custom-apps-documentation/app-components/rpcs/dynamic-fields-rpc.md
- https://developers.make.com/custom-apps-documentation/app-components/rpcs/dynamic-sample-rpc.md
- https://developers.make.com/custom-apps-documentation/best-practices/remote-procedure-calls.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/debugging-rpc.md
