# Publishing & Versioning

## App lifecycle stages

| Stage         | What it means                                                              |
| ------------- | -------------------------------------------------------------------------- |
| Draft (private) | Default. Visible only to your account; no diff tracking.                 |
| Published     | Made available for review and (after approval) to all Make users.          |
| Approved      | Code is locked; changes require review before being public.                |

> **Publishing is permanent.** You cannot un-publish.

## Pre-publish checklist

Source: https://developers.make.com/custom-apps-documentation/app-review/prerequisites.md

- Service is not already integrated by Make.
- Modules talk to a real REST/GraphQL API (no iterators/aggregators duplicating Make's primitives).
- API domain is owned by the service (NOT `heroku.com`, `amazonaws.com` subdomains).
- Only requests credentials the app actually uses.
- `log.sanitize` covers every secret.
- Base has a global error spec (401/403/404/422/429/500).
- Connection validates with wrong credentials (returns error, not silent success).
- Every module has correct label, description, attached connection.
- App includes a **Universal module** ("Make an API call").
- `interface.imljson` is complete; date fields typed as `date`.
- Search / trigger / RPC modules respect `parameters.limit` via `response.limit`.
- Pagination wherever API supports it.
- Test scenarios for every module, free of personal data.
- All best-practices in `best-practices.md` followed.

## App review process

1. Publish the app (in editor).
2. In Modules tab, toggle visibility per module — only visible modules go to review.
3. **Review** tab appears after publish.
4. Fill the form: API docs URL, public testing scenarios, vendor relationship, partnership contacts, categories, logo (ISV), service URL, trademark/ToS compliance.
5. Submit. Email thread `App review: <YourApp>` opens with Make.
6. Three phases:
   - **Form submission** (data collection)
   - **Automatic review (beta)** — checks + PDF feedback
   - **Manual review** — QA engineer reviews UX & code

7. On approval: app is available to all Make users.

## Versioning

- Make uses a single integer `appVersion` per app — not semver.
- For ongoing changes, edit in place; same version.
- For major API rewrites that would require breaking changes, **create a new app** (new app ID), deprecate the old.

## Testing vs Production pattern

1. Production app on Make.
2. Clone locally.
3. Create a separate empty testing app on Make (label `<App> Testing`).
4. Add as second origin in `makecomapp.json`.
5. Develop & test in the testing app's Scenario Builder.
6. Once validated, deploy to production origin.

See `tooling.md` for `makecomapp.json` origin schema.

## Updating an approved app

- Edits stay private to you until you request approval.
- Make tracks every change as a **diff**. **Changes** tab in app editor shows the log; "Show diff" reveals it.
- In VS Code, modified items have `*`; right-click → **Show changes**.
- Test new features with *Run once* / *Run this module only* before requesting review.
- Submit the changes for re-review via the same email thread.

## Breaking changes

What counts as breaking (avoid in approved apps unless properly managed):
- Any change to **base** or **custom IML functions**.
- OAuth `refresh` communication changes.
- Module response output / type / valid changes.
- Adding chained requests; swapping connection.
- Making a parameter required, removing it, narrowing its type, adding validation, removing options.
- Any webhook change.
- RPC parameter / dynamic-fields builder changes.

Non-breaking: safe type widening (number → text).

### Deprecation pattern (parameter)
1. Label: `[Deprecated] <field>`
2. Move to **Advanced parameters**
3. Help text: "Use <new field> instead"
4. (optional) Throw `DataError` after a grace period when the deprecated value is used

### Deprecation pattern (module)
1. Force a `valid` condition that always fails:
   ```json
   "valid": { "condition": "{{false}}", "message": "This module is deprecated. Use <new module>." }
   ```
2. Hide the module from new scenarios in the **Modules** tab.

### Deprecation pattern (connection)
1. Create new connection.
2. Rename old `... (deprecated)`.
3. Set new connection as the alternative on every module.

## Private apps & visibility

- Private apps (unapproved): all changes take effect immediately, no diff tool.
- Per-module **Hidden/Visible** toggle controls who sees the module in scenarios.
- For organization-restricted access, see: https://developers.make.com/custom-apps-documentation/community-apps/tips-and-tricks/control-of-access-in-apps-using-basic-connection.md

## Source URLs

- https://developers.make.com/custom-apps-documentation/app-review/overview.md
- https://developers.make.com/custom-apps-documentation/app-review/prerequisites.md
- https://developers.make.com/custom-apps-documentation/app-review/request-app-review.md
- https://developers.make.com/custom-apps-documentation/app-review/review-status.md
- https://developers.make.com/custom-apps-documentation/app-review/approved-app.md
- https://developers.make.com/custom-apps-documentation/app-maintenance/updating-your-app.md
- https://developers.make.com/custom-apps-documentation/app-maintenance/updating-your-app/approved-apps.md
- https://developers.make.com/custom-apps-documentation/app-maintenance/updating-your-app/approved-apps/tracking-code-changes.md
- https://developers.make.com/custom-apps-documentation/app-maintenance/updating-your-app/approved-apps/approval-of-changes-in-approved-app.md
- https://developers.make.com/custom-apps-documentation/app-maintenance/updating-your-app/approved-apps/managing-breaking-changes.md
- https://developers.make.com/custom-apps-documentation/app-maintenance/terms-of-approved-app-maintenance.md
