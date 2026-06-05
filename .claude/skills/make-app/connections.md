# Connections

A connection holds the credentials a user authorizes for the app. Per-connection folder: `connections/<name>/` with `metadata.json` + `communication.imljson` + `parameters.imljson` (+ `scope.imljson`, `scopes.imljson` for OAuth).

## Naming — connection IDs are server-generated

A connection has **two** identifiers:

| Identifier      | Where it lives                                 | Who picks it                                       |
| --------------- | ---------------------------------------------- | -------------------------------------------------- |
| **Local ID**    | Folder name `connections/<localId>/`           | Optional user input — empty = autogenerate         |
| **Remote name** | The actual ID in Make (used by API + modules)  | **Server-generated**: `<appId><N>` (incremental N) |

When a connection is first deployed, Make's backend assigns the remote name as the **app ID followed by an incrementing integer** — `1`, `2`, `3`, … globally across the app. For an app with `appId: "mfu-github-odpjbh"`:

- First connection deployed → `mfu-github-odpjbh1`
- Second connection → `mfu-github-odpjbh2`

The remote name is **immutable** once created. The repo folder name and the remote name can differ; under GitHub sync the folder↔remote pairing is tracked via the connection's `$id` (see below).

**Local ID validation** (when not empty): `^[a-zA-Z][0-9a-zA-Z-]{1,33}[0-9a-zA-Z]$` — 3-35 chars, must start with a letter, alphanumeric + dash, no trailing dash.

**Module references use the remote name.** Inside a module's `metadata.json`:
```json
"attachedAccounts": ["mfu-github-odpjbh1"]
```
…not the local folder name. Same applies for `webhook.connection` and `rpc.connection` references.

> When pulling/cloning an existing app, the repo folder name is usually the remote name (Make sends it back as-is). A folder rename is matched via the connection's `$id` (see below), which keeps `attachedAccounts`/`connection`/`altConnection` references intact across the sync.

### Stable identity across GitHub sync — `$id`

If the app is bound to a **GitHub repo**, the connection's `metadata.json` also carries a `$id` — a **UUID v4** that is the connection's stable identity for the repo↔Make sync, independent of both the (mutable) remote name and the repo folder name:

```json
{ "$id": "550e8400-e29b-41d4-a716-446655440000", "label": "GitHub OAuth Connection", "type": "oauth" }
```

A stable `$id` survives folder renames and lets the same repo be cloned into several apps (each app auto-names its own connection, all sharing the one `$id`). Make mints a random `$id` on first push if absent, but **provide your own UUID v4 when hand-authoring** the file so identity is deterministic from commit one. See `architecture.md` → "Stable component identity".

## Connection types — only two per schema

```json
{ "label": "GitHub OAuth Connection", "type": "oauth" }
{ "label": "Service API key",         "type": "basic" }
```

`type` ∈ **`"basic"`** or **`"oauth"`** — these are the only schema-valid values.

| Logical auth         | `type` value | Communication file shape                              |
| -------------------- | ------------ | ----------------------------------------------------- |
| Basic Auth, API Key, JWT, custom header tokens | `"basic"` | Single api.json request validating creds. |
| OAuth 2.0 (Auth Code / Client Credentials)     | `"oauth"` | api-oauth.json with `authorize`, `token`, `info`, `refresh`, `invalidate` phases. |
| OAuth 1.0                                       | `"oauth"` | api-oauth.json with `requestToken`, `authorize`, `accessToken`, `info` phases. |

> JWT is **not** a distinct type. It's a `basic` connection that builds the `Authorization: Bearer <jwt>` header dynamically using the `jwt()` IML function.

## Common patterns

- The connection's communication validates the credentials by hitting an endpoint that fails on wrong creds.
- Persist tokens / metadata into the connection via `response.data.<field>` — accessible in modules as `{{connection.<field>}}`.
- Set `response.uid` to the remote user ID (required for shared webhooks).
- Set `response.metadata` to a human label (shown in parentheses after the connection's user-given name).
- Place `common.client_id` / `common.client_secret` in Common Data (encrypted, locked after approval).
- Sanitize tokens, codes, secrets in every step's `log.sanitize`.

## `response.data` — persisting the token into the connection

The `data` directive **saves data to the connection** so it can be accessed later from any module through the `connection` variable. It works like the `temp` directive, **except `data` is persisted on the connection** (across module executions) instead of living only for the current request chain.

This is how a token obtained during connection validation (or token exchange) is stored once and reused by every module — the module never re-runs the auth call, it just reads `{{connection.<field>}}`.

**Save it** in the connection's `communication.imljson` (Basic) or the `token` / `info` / `refresh` phases (OAuth):
```json
{
	"response": {
		"data": {
			"accessToken": "{{body.token}}"
		}
	}
}
```

**Use it later** in `base.imljson` (inherited by every module request):
```json
{
	"url": "https://example.com",
	"headers": {
		"X-API-Key": "{{connection.accessToken}}"
	}
}
```

Notes:
- Each `response.data` key becomes a `{{connection.<key>}}` field — e.g. `data.accessToken` → `{{connection.accessToken}}`.
- For OAuth, also store `refreshToken` and `expires` in `data` so the `refresh` phase can renew silently (see the OAuth2 example below).
- The token is secret: add the source field to `log.sanitize` (e.g. `response.body.token`, `request.headers.\`X-API-Key\``).
- `connection.*` is read-only outside the connection — to update a stored value, re-set it via `response.data` in a connection phase (e.g. `refresh`).

## Basic connection (API Key, Basic Auth, custom token)

### `parameters.imljson`
```json
[
  {
    "name": "apiKey",
    "type": "password",
    "label": "API Key",
    "help": "Find at https://example.com/settings/api",
    "required": true
  }
]
```

### `communication.imljson` — validates the key
```json
{
  "url": "https://api.example.com/v1/me",
  "headers": { "x-api-key": "{{parameters.apiKey}}" },
  "response": {
    "uid": "{{body.id}}",
    "metadata": { "type": "text", "value": "{{body.email}}" },
    "data": { "apiKey": "{{parameters.apiKey}}" }
  },
  "log": { "sanitize": ["request.headers.`x-api-key`"] }
}
```

Stored as `{{connection.apiKey}}` — use in base headers/qs.

## OAuth 2.0 (Authorization Code)

### `parameters.imljson` — optional, allows per-user client overrides
```json
[
  { "name": "clientId", "type": "text", "label": "Client ID", "help": "Your OAuth app's Client ID." },
  { "name": "clientSecret", "type": "password", "label": "Client Secret" },
  { "name": "scopes", "type": "array", "label": "Additional scopes", "spec": { "type": "text" } }
]
```

### `scope.imljson` — default scopes the connection always requests
```json
["read:user", "repo"]
```

### `scopes.imljson` — descriptions for every scope (UI display)
```json
{
  "read:user": "Read the authenticated user's profile",
  "repo": "Full control of private repositories",
  "public_repo": "Access to public repositories"
}
```

### `communication.imljson` — full OAuth2 flow (GitHub example)
```json
{
  "authorize": {
    "url": "https://github.com/login/oauth/authorize",
    "qs": {
      "client_id": "{{ifempty(parameters.clientId, common.clientId)}}",
      "redirect_uri": "{{oauth.redirectUri}}",
      "scope": "{{join(distinct(merge(oauth.scope, ifempty(parameters.scopes, emptyarray))), ' ')}}",
      "response_type": "code"
    },
    "response": { "temp": { "code": "{{query.code}}" } }
  },

  "token": {
    "url": "https://github.com/login/oauth/access_token",
    "method": "POST",
    "type": "urlencoded",
    "body": {
      "code": "{{temp.code}}",
      "client_id": "{{ifempty(parameters.clientId, common.clientId)}}",
      "client_secret": "{{ifempty(parameters.clientSecret, common.clientSecret)}}",
      "grant_type": "authorization_code",
      "redirect_uri": "{{oauth.redirectUri}}"
    },
    "response": {
      "data": {
        "accessToken": "{{body.access_token}}",
        "refreshToken": "{{body.refresh_token}}",
        "expires": "{{addSeconds(now, body.expires_in)}}"
      }
    },
    "log": { "sanitize": ["request.body.code", "request.body.client_secret", "response.body.access_token", "response.body.refresh_token"] }
  },

  "info": {
    "url": "https://api.github.com/user",
    "headers": { "authorization": "Bearer {{connection.accessToken}}" },
    "response": {
      "uid": "{{body.id}}",
      "data": { "login": "{{body.login}}" },
      "metadata": { "type": "text", "value": "{{body.login}}" }
    },
    "log": { "sanitize": ["request.headers.authorization"] }
  },

  "refresh": {
    "condition": "{{data.expires < addMinutes(now, 15)}}",
    "url": "https://github.com/login/oauth/access_token",
    "method": "POST",
    "type": "urlencoded",
    "body": {
      "grant_type": "refresh_token",
      "refresh_token": "{{data.refreshToken}}",
      "client_id": "{{ifempty(parameters.clientId, common.clientId)}}",
      "client_secret": "{{ifempty(parameters.clientSecret, common.clientSecret)}}"
    },
    "response": {
      "data": {
        "accessToken": "{{body.access_token}}",
        "refreshToken": "{{body.refresh_token}}",
        "expires": "{{addSeconds(now, body.expires_in)}}"
      }
    },
    "log": { "sanitize": ["request.body.refresh_token", "request.body.client_secret", "response.body.access_token", "response.body.refresh_token"] }
  },

  "invalidate": {
    "url": "https://api.github.com/applications/{{ifempty(parameters.clientId, common.clientId)}}/token",
    "method": "DELETE",
    "headers": { "authorization": "Basic {{base64(ifempty(parameters.clientId, common.clientId) + ':' + ifempty(parameters.clientSecret, common.clientSecret))}}" },
    "body": { "access_token": "{{connection.accessToken}}" }
  }
}
```

### OAuth2 phases
| Phase          | When it runs                                        | Purpose                                           |
| -------------- | --------------------------------------------------- | ------------------------------------------------- |
| `preauthorize` | Before redirect                                     | Optional, e.g. dynamically build the auth URL.    |
| `authorize`    | User clicks "Authorize" — Make redirects to service | Defines redirect URL (with `client_id` & scope).  |
| `token`        | After redirect back with `code`                     | Exchanges code → access token.                    |
| `info`         | On every connection use (and after token)           | Validates token; sets `uid` / `metadata` / data.  |
| `refresh`      | When `condition` is true                            | Silent token refresh.                             |
| `invalidate`   | On connection delete                                | Revokes the token at the provider.                |

### Callback URL
- OAuth 2 callback: `https://www.make.com/oauth/cb/app`
- OAuth 1 callback: `https://www.integromat.com/oauth/cb/app-oauth1`

Configure the same URL in the provider's OAuth app settings.

### Common Data (`common.*`)
Holds `client_id` and `client_secret` — encrypted, locked after approval, shared across all installs of the app. Access via `{{common.clientId}}`.

```json
{ "clientId": "abc123", "clientSecret": "shhhh" }
```

## OAuth 2.0 (Client Credentials)

### `communication.imljson`
```json
{
  "token": {
    "url": "https://api.example.com/oauth/token",
    "method": "POST",
    "type": "urlencoded",
    "body": {
      "grant_type": "client_credentials",
      "client_id": "{{parameters.clientId}}",
      "client_secret": "{{parameters.clientSecret}}",
      "scope": "{{join(oauth.scope, ' ')}}"
    },
    "response": {
      "data": {
        "accessToken": "{{body.access_token}}",
        "expires": "{{addSeconds(now, body.expires_in)}}"
      }
    },
    "log": { "sanitize": ["request.body.client_secret", "response.body.access_token"] }
  },
  "info": {
    "url": "https://api.example.com/v1/me",
    "headers": { "authorization": "Bearer {{connection.accessToken}}" },
    "response": {
      "metadata": { "type": "text", "value": "{{body.organization}}" }
    },
    "log": { "sanitize": ["request.headers.authorization"] }
  }
}
```

## JWT connection (implemented as Basic)

`metadata.json`:
```json
{ "label": "Service Account", "type": "basic" }
```

`parameters.imljson`:
```json
[
  { "name": "email", "type": "text", "label": "Service account email", "required": true },
  { "name": "privateKey", "type": "pkey", "label": "Private key (PEM)", "required": true }
]
```

`communication.imljson`:
```json
{
  "url": "https://api.example.com/v1/me",
  "headers": {
    "authorization": "Bearer {{jwt({iss: parameters.email, scope: 'read', aud: 'https://api.example.com', exp: addMinutes(now, 30), iat: now}, parameters.privateKey, 'RS256')}}"
  },
  "response": {
    "data": { "email": "{{parameters.email}}", "privateKey": "{{parameters.privateKey}}" },
    "metadata": { "type": "email", "value": "{{parameters.email}}" }
  },
  "log": { "sanitize": ["request.headers.authorization"] }
}
```

Reuse via `{{jwt({...}, connection.privateKey, 'RS256')}}` in base headers.

## OAuth 1.0

Rare. Phases: `requestToken` → `authorize` → `accessToken` → `info`. Use the `oauth` directive in base/communication:
```json
{
  "oauth": {
    "consumer_key":    "{{parameters.consumerKey}}",
    "consumer_secret": "{{parameters.consumerSecret}}",
    "token":           "{{connection.token}}",
    "token_secret":    "{{connection.tokenSecret}}",
    "signature_method": "HMAC-SHA1",
    "transport_method": "header"
  }
}
```

## Editable vs locked parameters

```json
{ "name": "apiKey", "type": "password", "label": "API Key", "required": true, "editable": true }
```

`editable: true` — user can update without recreating the connection.
**Never** allow editing for subdomain/URL parameters (SSRF risk).

## Reserved parameter names — do NOT use

- `accountName` (connection name)
- `teamID` (assigned team)

## Source URLs

- https://developers.make.com/custom-apps-documentation/app-components/connections.md
- https://developers.make.com/custom-apps-documentation/app-components/connections/basic-connection.md
- https://developers.make.com/custom-apps-documentation/app-components/connections/oauth2.md
- https://developers.make.com/custom-apps-documentation/app-components/connections/oauth1.md
- https://developers.make.com/custom-apps-documentation/app-components/connections/jwt.md
- https://developers.make.com/custom-apps-documentation/best-practices/connections.md
- https://developers.make.com/custom-apps-documentation/best-practices/connections/editable-connection.md
- https://developers.make.com/custom-apps-documentation/best-practices/connections/connection-metadata.md
- https://developers.make.com/custom-apps-documentation/best-practices/connections/additional-oauth-scopes.md
