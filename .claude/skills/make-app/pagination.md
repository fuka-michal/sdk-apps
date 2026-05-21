# Pagination

The `pagination` collection sits at the request level (next to `url`/`method`/`response`) of Search modules, polling triggers, and dynamic-options RPCs.

## Schema

| Key                | Type                   | Default | Purpose                                       |
| ------------------ | ---------------------- | ------- | --------------------------------------------- |
| `url`              | IML string             | —       | Override URL for next page.                   |
| `method`           | IML string             | —       | Override method for next page.                |
| `qs`               | flat object            | —       | Merged into next request's query string.      |
| `headers`          | flat object            | —       | Merged into next request's headers.           |
| `body`             | any                    | —       | Replaces next request's body (non-GET).       |
| `condition`        | IML string \| boolean  | —       | Truthy → fetch another page; falsy → stop.    |
| `mergeWithParent`  | boolean                | `true`  | If `false`, replaces parent params entirely.  |

Inside `pagination` scope, the auto-counter `pagination.page` is 1-based.

## Patterns

### Page number
```json
"pagination": {
  "qs": { "page": "{{pagination.page}}" },
  "condition": "{{body.totalPages >= pagination.page}}"
}
```

### Limit + offset
```json
"pagination": {
  "qs": { "offset": "{{(pagination.page - 1) * 100}}" }
}
```
(Make stops when an empty page returns and no `condition` exists; better to add `"condition": "{{length(body.data) > 0}}"`.)

### Cursor / next-page token
```json
"pagination": {
  "qs": { "cursor": "{{body.next_cursor}}" },
  "condition": "{{body.next_cursor}}"
}
```

### Next-URL link (HATEOAS)
```json
"pagination": {
  "url": "{{body._links.next.href}}",
  "condition": "{{body._links.next.href}}"
}
```

### Has-more boolean
```json
"pagination": {
  "qs": { "page": "{{pagination.page}}" },
  "condition": "{{body.has_more}}"
}
```

### Cursor saved across requests via `temp`
```json
"response": {
  "temp": { "cursor": "{{body.next_cursor}}" }
},
"pagination": {
  "qs": { "cursor": "{{temp.cursor}}" },
  "condition": "{{temp.cursor}}"
}
```

### Link header (e.g., GitHub)
```json
"pagination": {
  "url": "{{headers.`link`}}",
  "condition": "{{contains(headers.`link`, 'rel=\"next\"')}}"
}
```

(GitHub returns `Link` header; use a custom IML function `extractNextLink(headers.link)` for clean parsing.)

## Limits

| Limit                                | Value         |
| ------------------------------------ | ------------- |
| Max total HTTP requests per module   | 100           |
| Max pagination requests              | 50            |
| Max paginated records                | 3,200         |
| Max module execution time            | 40 seconds    |

If your search needs >3200 records, the result set is silently truncated. Document in the module description.

## Honoring user limits

Always respect `parameters.limit` via `response.limit`:
```json
"response": { "limit": "{{parameters.limit}}", "iterate": "{{body.data}}", "output": "{{item}}" }
```

Make stops paginating once enough items are collected — no extra HTTP calls.

## RPCs

In dynamic-options RPCs, cap the page count explicitly:
```json
"pagination": {
  "qs": { "page": "{{pagination.page}}" },
  "condition": "{{pagination.page <= 3}}"
}
```

3 pages × 100 items = 300 options — sane upper bound for select dropdowns.

## Polling triggers

Use the maximum page size the API allows. Sort DESC so Make stops at the first known item. See `modules.md` → Polling Trigger.

```json
{
  "url": "/events",
  "qs": { "sort": "desc", "per_page": 100 },
  "pagination": { "qs": { "page": "{{pagination.page}}" } },
  "response": {
    "iterate": "{{body.events}}",
    "trigger": { "type": "date", "order": "desc", "id": "{{item.id}}", "date": "{{item.created_at}}" },
    "output":  "{{item}}",
    "limit":   "{{parameters.limit}}"
  }
}
```

## Common bugs

| Symptom                                  | Likely cause                                                  |
| ---------------------------------------- | ------------------------------------------------------------- |
| User's `limit` ignored                   | Missing `response.limit`                                      |
| Only first page returned                 | `condition` evaluates falsy too early                         |
| One extra empty-page request             | `condition` doesn't check current-page result                 |
| Module returns more than requested       | `response.limit` references wrong variable                    |
| Duplicates                               | Overlapping ranges (e.g., offset + sort changes between pages)|
| Hits 50/3200 cap without warning         | Add a `pagination.condition` to stop early                    |

## Source URLs

- https://developers.make.com/custom-apps-documentation/component-blocks/api/pagination.md
- https://developers.make.com/custom-apps-documentation/debug-your-app/debugging-of-pagination-in-list-search-modules.md
