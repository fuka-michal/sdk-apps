# Tooling — VS Code Extension & Make CLI

Two official ways to develop Make apps locally: the **VS Code "Make Apps Editor" extension** (preferred for daily work) and the **Make CLI** (scriptable, CI/CD).

## VS Code "Make Apps Editor"

- Marketplace: https://marketplace.visualstudio.com/items?itemName=Integromat.apps-sdk
- Source: https://github.com/integromat/vscode-apps-sdk
- Two modes: **Online** (cloud-edit) and **Local** (offline, beta).

### Install
Search "Make Apps Editor" (publisher: Integromat) in VS Code Extensions. Make icon appears in the sidebar.

### Recommended VS Code settings
```json
{
  "editor.formatOnSave": true,
  "editor.quickSuggestions.strings": true
}
```

### Configure environment
1. Generate Make API key: Profile → API → **Add token**. Required scopes: `sdk-apps:read`, `sdk-apps:write`. (Copy immediately — Make only shows it once.)
2. Click Make icon → **Add environment** or palette `>Make Apps: Add SDK Environment`.
3. API URL (zone-specific):
   - EU1: `eu1.make.com/api`
   - EU2: `eu2.make.com/api`
   - US1: `us1.make.com/api`
4. Label the environment, paste the API key.
5. Switch zones from status bar; `>Login` / `>Logout` from palette.

### Create new app
- `+` in **My apps**, or palette `>New app`.
- Inputs: label → app ID (regex `^[a-z][0-9a-z-]+[0-9a-z]$`, **immutable**) → version (currently 1) → description → theme hex → language → countries (empty = global).

### Online mode
- Each `.imljson` opens as a temporary file.
- Saving uploads to Make automatically; file auto-deletes on close.

## Make CLI (`@makehq/cli`)

Separate official terminal tool. Useful for CI/CD or scripted bulk edits. Not a full replacement for the VS Code extension's local mode.

### Install

```bash
npm install -g @makehq/cli
# or
brew install make-cli integromat/tap/make-cli
# or
npx @makehq/cli <command>
```

Binaries: https://github.com/integromat/make-cli/releases

### Auth

```bash
make-cli login              # wizard: pick zone, paste API key
make-cli whoami
make-cli logout
```

Config saved at:
- macOS / Linux: `~/.config/make-cli/config.json`
- Windows: `%APPDATA%\make-cli\config.json`

Env vars override saved login:
```bash
export MAKE_API_KEY="..."
export MAKE_ZONE="eu1"
```

### Apps

```bash
make-cli sdk-apps list
make-cli sdk-apps get          --name <id> --version <n>
make-cli sdk-apps create       --name <id> --label "<text>" --theme "#hex" --language en --audience global
make-cli sdk-apps update       --name <id> --version <n> [--label ...] [--description ...]
make-cli sdk-apps delete       --name <id> --version <n>
make-cli sdk-apps get-section  --name <id> --version <n> --section <base|groups|readme|common>
make-cli sdk-apps set-section  --name <id> --version <n> --section <section> --body <jsonc>
make-cli sdk-apps get-docs     --name <id> --version <n>
make-cli sdk-apps set-docs     --name <id> --version <n> --docs <markdown>
make-cli sdk-apps get-common   --name <id> --version <n>
make-cli sdk-apps set-common   --name <id> --version <n> --common <json>
```

### Connections

```bash
make-cli sdk-connections list   --app-name <id>
make-cli sdk-connections create --app-name <id> --type oauth2 --label "<text>"
make-cli sdk-connections get-section --connection-name <id> --section communication
make-cli sdk-connections set-section --connection-name <id> --section communication --body <jsonc>
make-cli sdk-connections set-common  --connection-name <id> --common <json>
```

### Modules

```bash
make-cli sdk-modules list   --app-name <id> --app-version <n>
make-cli sdk-modules create --app-name <id> --app-version <n> --name <id> --type-id <n> --label "<text>" --description "<text>"
make-cli sdk-modules get-section --app-name <id> --app-version <n> --module-name <id> --section communication
make-cli sdk-modules set-section --app-name <id> --app-version <n> --module-name <id> --section parameters --body <jsonc>
```

### RPCs

```bash
make-cli sdk-rpcs list   --app-name <id> --app-version <n>
make-cli sdk-rpcs create --app-name <id> --app-version <n> --name <id> --label "<text>"
make-cli sdk-rpcs test   --app-name <id> --app-version <n> --rpc-name <id> --data <json> --schema <json>
make-cli sdk-rpcs set-section --app-name <id> --app-version <n> --rpc-name <id> --section communication --body <jsonc>
```

### Webhooks

```bash
make-cli sdk-webhooks list   --app-name <id>
make-cli sdk-webhooks create --app-name <id> --type shared --label "<text>"
make-cli sdk-webhooks set-section --webhook-name <id> --section communication --body <jsonc>
```

### Functions

```bash
make-cli sdk-functions list   --app-name <id> --app-version <n>
make-cli sdk-functions create --app-name <id> --app-version <n> --name <id>
make-cli sdk-functions get-code  --app-name <id> --app-version <n> --function-name <id>
make-cli sdk-functions set-code  --app-name <id> --app-version <n> --function-name <id> --code <jscode>
make-cli sdk-functions set-test  --app-name <id> --app-version <n> --function-name <id> --test <jscode>
```

Section bodies use **JSONC** (JSON with comments). The CLI has no bulk push/pull; script per section, or use VS Code local mode.

## Engine version

Currently only `engineVersion: 1` is exposed for newly created apps. Legacy v2 apps continue to run but new apps cannot opt into v2.

## Source URLs

- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/configuration-of-vs-code.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/generation-of-your-api-key.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/creating-an-app-in-vs-code.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/local-development-for-apps.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/local-development-for-apps/clone-make-app-to-local-workspace.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/local-development-for-apps/develop-app-in-a-local-workspace-offline.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/local-development-for-apps/deploy-changes-from-local-app-to-make-app.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/local-development-for-apps/pull-changes-from-make-app.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/local-development-for-apps/compare-changes-between-local-and-make-app.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/local-development-for-apps/create-a-new-app-origin.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/manage-testing-and-production-app-versions.md
- https://developers.make.com/make-cli/make-cli.md
- https://developers.make.com/make-cli/make-cli/install-the-make-cli.md
- https://developers.make.com/make-cli/make-cli/authenticate-the-make-cli.md
- https://developers.make.com/make-cli/make-cli/make-cli-reference.md
- https://developers.make.com/make-cli/make-cli/make-cli-reference/custom-app-development.md
