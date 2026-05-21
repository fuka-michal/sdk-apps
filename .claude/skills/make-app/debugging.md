# Debugging

Three layers of debugging:
1. **Make UI scenario testing** (Run once / Run this module only)
2. **Make DevTool** (Chrome extension)
3. **Browser console + `debug()`** (for custom IML functions)

## Testing in Make UI

1. In a scenario, add your module.
2. Click **Create a connection** if not already configured.
3. Fill the mappable params with known-good values.
4. **Run once** at the bottom-left.
5. Inspect output: `Output → Bundle 1 → response → ...`.

New private modules show a **Private** tag; toggle visibility in app settings.

## Make DevTool (Chrome extension)

Install: Chrome Web Store → search "Make DevTool".

Usage: open a scenario page → DevTools (Cmd+Opt+I) → **Make** tab.

Three views:

### Live Stream
Appears after **Run once**. Per module shows:
- Request: headers, method, URL, query string
- Request body
- Response: headers, status, body
- Search bar, clear (trash), console-log toggle, **Copy RAW** (JSON) and **Copy cURL** buttons.

### Scenario Debugger
Historical run inspector. Search modules by name/ID, double-click to open config, click operations to inspect request/response.

### Tools (15 helpers)
- Focus a Module, Find Modules by Mapping, Get App Metadata
- Copy Mapping / Filter
- Swap Connection / Variable / App
- Base64 encode/decode
- Get Blueprint Size, Showcase Mode, Mock Labels & Change Background, Remap Source, Highlight App

## Debugging custom IML functions

Use `debug(...)` (NOT `console.log`). Output appears in the browser DevTools console while a scenario runs.

```javascript
function buildPayload(params) {
    debug('input: ' + JSON.stringify(params));
    const cleaned = iml.removeEmpty(params);
    debug(`cleaned: ${JSON.stringify(cleaned)}`);
    return cleaned;
}
```

Open the browser console:
- macOS: Cmd+Opt+J
- Win/Linux: Ctrl+Shift+J

For pure JavaScript dry-runs (no Make context), paste into RunJS / MDN Playground.

## Debugging RPCs

1. Open Make → Custom apps → your app → **Remote Procedures** tab.
2. Select RPC → **Test RPC**.
3. Specify connection and parameters → **Test** → inspect output.
4. For raw HTTP, use the browser's Network tab.

Tips:
- Empty arrays → wrong `iterate` path (try `body` vs `body.data` vs `body.items`).
- Select RPC must return only `{label, value}` shape — extra fields cause issues.
- Use the RPC's `parameters.imljson` to expose debug-only fields not shown in modules.

## Debugging pagination

Seed >1 page of records:
- Use Flow Control > **Repeater** to create N test records, each with a unique ID.
- Set the API endpoint's `per_page` low (e.g., 5) to force pagination quickly.

Watch the Live Stream — each pagination request appears as a separate row. Common bugs:

| Symptom                           | Fix                                                    |
| --------------------------------- | ------------------------------------------------------ |
| One page only                     | `pagination.condition` falsy on first page             |
| Extra empty page after last       | Condition doesn't check current-page result            |
| User limit ignored                | Missing `response.limit`                               |
| Way too many small pages          | Increase `per_page`                                    |
| Duplicates                        | Offset-based with changing sort — switch to cursor     |

## IML tests

In VS Code: right-click function name → **Run IML test**. See `iml-functions.md`.

For timezone-sensitive tests, set the extension's timezone (e.g. `Europe/Prague`).

## `log.sanitize` and observability

Without `log.sanitize`, Make hides logs entirely from DevTool — meaning you can't see your request at all.

Always sanitize, then logs become visible:
```json
"log": { "sanitize": ["request.headers.authorization", "request.qs.api_key"] }
```

## Common gotchas

- **404 from Make DevTool**: Make's edge stripped a header. Inspect raw cURL via Copy cURL button.
- **`body` is empty in IML**: response was empty or wrong `type` (parse failed). Try `"type": "text"` to see raw body.
- **`{{item}}` is undefined inside `iterate.output`**: `iterate` path is wrong; `body.data` vs `body.items` vs `body`.
- **Module saves but module errors with `Internal error`**: usually invalid JSON in `interface.imljson` (`type` mismatch) or unsanitized log.
- **`{{connection.x}}` is undefined**: connection didn't write to `response.data.x` in `info` or `token`.
- **Trigger runs but never emits**: descending sort missing, or `trigger.id` is the same as the last cursor.

## Source URLs

- https://developers.make.com/custom-apps-documentation/debug-your-app/overview.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/make-devtool.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/make-devtool/live-stream.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/make-devtool/scenario-debugger.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/make-devtool/tools.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/debugging-rpc.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/debugging-of-custom-iml-functions.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/debugging-of-pagination-in-list-search-modules.md
- https://developers.make.com/custom-apps-documentation/create-your-first-app/test-your-app.md
