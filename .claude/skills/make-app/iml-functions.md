# Custom IML Functions

JavaScript helpers in `functions/<name>/code.js` accessible from any `.imljson` file. Used when an IML one-liner isn't enough (multi-step transforms, schema conversion, removing nulls).

> Custom functions are **not available by default** — request the feature via Make helpdesk for your app.

## Folder structure

```
functions/
├── <name>/
│   ├── code.js           function <name>(args) { ... return ...; }
│   └── test/
│       └── test.js       it("...", () => { assert.strictEqual(...) })
```

The function's **name** in `code.js` must match the folder name.

## Sandbox

| Available                              | Not available                       |
| -------------------------------------- | ----------------------------------- |
| ES6+ syntax                            | `require()` / `import`              |
| Built-in JavaScript objects            | `fs`, `process`, network            |
| `Buffer`                               | External npm modules                |
| `iml.*` (call other IML functions)     | `console.log` — use `debug()`       |
| `debug(...)` (DevTool console output)  | Anything beyond the sandbox         |

| Limit                | Value           |
| -------------------- | --------------- |
| Max execution time   | 10 seconds      |
| Max source length    | 5,000 chars     |

## Calling from `.imljson`

```json
{ "body": "{{myFunction(parameters.input)}}" }
```

Functions can call each other via `iml.<name>(...)`:
```javascript
function buildPayload(data) {
    const cleaned = iml.removeEmpty(data);
    return iml.formatDate(cleaned.created, 'YYYY-MM-DD');
}
```

## Common useful functions

### `removeEmpty` — drop null/undefined/empty entries

```javascript
function removeEmpty(obj) {
    if (Array.isArray(obj)) {
        return obj.map(removeEmpty).filter(v => v !== undefined && v !== null);
    }
    if (obj && typeof obj === 'object') {
        const out = {};
        for (const k of Object.keys(obj)) {
            const v = removeEmpty(obj[k]);
            if (v === undefined || v === null) continue;
            if (typeof v === 'string' && v === '') continue;
            if (Array.isArray(v) && v.length === 0) continue;
            if (typeof v === 'object' && !Array.isArray(v) && Object.keys(v).length === 0) continue;
            out[k] = v;
        }
        return out;
    }
    return obj;
}
```

Use in module body:
```json
{ "body": "{{removeEmpty(parameters)}}" }
```

Or inline-merge with a fixed shape:
```json
{ "body": { "id": "{{parameters.id}}", "{{...}}": "{{removeEmpty(parameters.extra)}}" } }
```

### `dynamicFields` — convert API field schema → Make collection

```javascript
function dynamicFields(fields) {
    if (!fields) return;
    const out = [];
    fields.forEach(f => {
        const p = { name: f.id, label: f.name, required: !!f.required };
        switch (f.type) {
            case 'date':
            case 'anniversary':
                p.type = 'date'; break;
            case 'multi_line_text':
                p.multiline = true;
                p.type = 'text'; break;
            case 'single_line_text':
                p.type = 'text'; break;
            case 'number':
                p.type = 'number'; break;
            case 'multiple_choice':
                p.multiple = true;
                p.type = 'select';
                p.options = (f.choices || []).map(c => ({ label: c, value: c }));
                break;
            case 'select_box':
                p.type = 'select';
                p.options = (f.choices || []).map(c => ({ label: c, value: c }));
                break;
            default:
                return;
        }
        out.push(p);
    });
    return { name: 'data', label: 'Data', type: 'collection', spec: out };
}
```

Then in an RPC:
```json
{ "response": { "output": "{{dynamicFields(body.fields)}}" } }
```

### `buildHeaders` — dynamic headers builder

```javascript
function buildHeaders(parameters) {
    const headers = { 'content-type': 'application/json' };
    if (parameters.tenant)  headers['x-tenant-id']  = parameters.tenant;
    if (parameters.locale)  headers['accept-language'] = parameters.locale;
    return headers;
}
```

Use to replace base headers:
```json
{ "headers": "{{buildHeaders(parameters)}}" }
```

To merge with base headers instead:
```json
{ "headers": { "{{...}}": "{{buildHeaders(parameters)}}" } }
```

## Tests — `functions/<name>/test/test.js`

Mocha-style with global `it` and Node's `assert`:

```javascript
it('removes nulls and empty strings', () => {
    const result = removeEmpty({ a: 'x', b: null, c: '', d: { e: undefined, f: 'keep' } });
    assert.deepStrictEqual(result, { a: 'x', d: { f: 'keep' } });
});

it('handles arrays of empties', () => {
    assert.deepStrictEqual(removeEmpty([null, '', 'ok', undefined]), ['ok']);
});

it('returns undefined for null input', () => {
    assert.strictEqual(removeEmpty(null), null);
});
```

Available assertions:
| Method                              | Purpose                  |
| ----------------------------------- | ------------------------ |
| `assert.ok(x)`                      | Truthy                   |
| `assert.strictEqual(a, b)`          | `===`                    |
| `assert.deepStrictEqual(a, b)`      | Deep equality            |
| `assert.notStrictEqual(a, b)`       | `!==`                    |
| `assert.throws(fn, [err])`          | Function throws          |
| `assert.doesNotThrow(fn)`           | Function does not throw  |
| `assert.match(s, regex)`            | Regex match              |
| `assert.doesNotMatch(s, regex)`     | Regex non-match          |

Run from VS Code: right-click function → **Run IML test**. Output appears in IML tests channel.

For timezone-sensitive tests, set the VS Code extension's timezone setting (e.g. `Europe/Prague`).

## Debugging

Use `debug(...)` (NOT `console.log`) — output appears in Make DevTool's console.

```javascript
function add(a, b) {
    const sum = a + b;
    debug(`add(${a}, ${b}) = ${sum}`);
    return sum;
}
```

For pure JS dry-runs, paste into RunJS / MDN Playground.

## Style rules

- Pure functions only. No global state.
- One responsibility per function.
- Keep functions under 5000 characters of source.
- Use `iml.<name>` to compose built-in helpers — don't re-implement `formatDate`, `parseDate`, `base64`, etc.

## Source URLs

- https://developers.make.com/custom-apps-documentation/app-components/iml-functions.md
- https://developers.make.com/custom-apps-documentation/app-components/iml-functions/dynamic-mappable-parameters.md
- https://developers.make.com/custom-apps-documentation/app-components/iml-functions/removal-of-empty-collections-and-nulls.md
- https://developers.make.com/custom-apps-documentation/app-components/iml-functions/handling-of-full-update-approach-in-update-modules.md
- https://developers.make.com/custom-apps-documentation/get-started/make-apps-editor/apps-sdk/iml-tests.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/debugging-of-custom-iml-functions.md
