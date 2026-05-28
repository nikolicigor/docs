---
name: cross-repo-sync
description: Verify that docs claims match the actual BE / SDK / FE implementation. Use whenever docs are edited or before a release. Catches the most common docs-bug — "docs say X, code does Y".
disable-model-invocation: false
---

## Cross-repo sync (docs perspective)

Docs lie when code changes and docs don't. This skill walks the surfaces and flags drift.

### Step 1 — Diff the docs

```!
git diff --stat main...HEAD
```

For each changed `.mdx`, identify which BE / SDK / FE surface it describes.

### Step 2 — BE check

For every endpoint mentioned in changed pages:

```bash
# Confirm the path exists in the BE
rg "@Controller\|@(Get|Post|Patch|Delete|Put)" /Users/Nikolic-Marko/FileFeed/filefeed-backend/src/modules --type ts | grep -i "<path-fragment>"
```

- Does the path actually exist in a controller?
- Does the method match (GET / POST / PATCH / …)?
- For deprecated endpoints: is `@DeprecatedInVersion` actually applied?
- For request/response shape: does the DTO under `src/modules/*/dto/` match what the docs claim?

### Step 3 — SDK check

For every SDK example in changed pages:

```bash
# Confirm the SDK method exists
rg "async (list|retrieve|create|update|remove|test)\(" /Users/Nikolic-Marko/FileFeed/filefeed-backend/filefeed-sdk/src/resources/ --type ts
```

- Does `filefeed.X.Y()` exist?
- Does the method signature match what the docs example shows?
- Is the SDK version mentioned in docs the currently published version?

### Step 4 — FE check (if user-flow docs)

If the docs describe a dashboard action ("click Settings → API → upgrade"):

- Does the dashboard route exist? (`/Users/Nikolic-Marko/FileFeed/filefeed-frontend/src/app/(dashboard)/`)
- Are the labels in the docs the actual UI labels? (case-sensitive — "API Version" vs "API version")
- Are screenshots current? (file dates >12 months old in `images/` = audit candidate)

### Step 5 — Changelog check

```!
git diff main...HEAD -- changelog.mdx | head -40
```

For every new endpoint / SDK method / deprecation introduced in this PR — is there a `changelog.mdx` entry under the right version date?

For every BE / SDK PR merged since the last docs deploy — is its change reflected in changelog?

### Step 6 — Migration guide check

If this PR adds or modifies `migration/*.mdx`:

- Before/after examples compile against the SDK versions involved?
- Curl examples use the correct `FileFeed-Version` header?
- Linked from `changelog.mdx` and from the relevant resource page?

### Step 7 — Report

```
| Surface       | Status | Detail                                         |
| ------------- | ------ | ---------------------------------------------- |
| BE endpoints  | ✅     | All paths mentioned exist in controllers       |
| SDK methods   | ⚠️     | docs/api-reference/connections.mdx mentions `rotate()` — SDK has `rotateCredentials()`. Pick one. |
| FE flows      | ✅     | Settings → API page exists; copy matches       |
| changelog.mdx | ❌     | New rotate endpoint not in changelog          |
| openapi.json  | ✅     | path entry added                              |
```

Anything ⚠️ or ❌ blocks merge.

### Forbidden

- Marking ✅ without actually running the corresponding `rg` against the sibling repo
- Adding a code example without verifying it would actually compile/run
- Skipping the changelog entry because "it's a small change"
