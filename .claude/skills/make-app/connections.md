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

The remote name is **immutable** once created. The VS Code extension stores the local↔remote pairing inside `origins[].idMapping.connection` in `makecomapp.json`:

```json
{
  "origins": [{
    "appId": "mfu-github-odpjbh",
    "idMapping": {
      "connection": [
        { "local": "githubOauth", "remote": "mfu-github-odpjbh1" }
      ]
    }
  }]
}
```

**Local ID validation** (VS Code extension, when not empty): `^[a-zA-Z][0-9a-zA-Z-]{1,33}[0-9a-zA-Z]$` — 3-35 chars, must start with a letter, alphanumeric + dash, no trailing dash.

**Module references use the remote name.** Inside a module's `metadata.json`:
```json
"attachedAccounts": ["mfu-github-odpjbh1"]
```
…not the local folder name. Same applies for `webhook.connection` and `rpc.connection` references.

> When pulling/cloning an existing app, the local folder name is usually the remote name (Make sends it back as-is). Renaming the folder requires updating `idMapping` and every `attachedAccounts`/`connection`/`altConnection` reference.

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
