---
name: pre-merge-check
description: Run the docs pre-merge gate — JSON validation, dead-link scan, Mintlify build check, cross-repo sync. Use before requesting PR review on the docs site.
disable-model-invocation: false
---

## Docs pre-merge gate

The docs site is a customer-facing artifact; broken pages mean broken integrations.

### Step 1 — Working tree

```!
git status --short
git branch --show-current
```

Branch must be `feat/*`, `fix/*`, `chore/*`, or `docs/*`. Never `main`.

### Step 2 — JSON validity

The two JSON files are the most fragile:

```!
node -e "JSON.parse(require('fs').readFileSync('api-reference/openapi.json','utf8')); console.log('openapi.json: valid')"
node -e "JSON.parse(require('fs').readFileSync('docs.json','utf8')); console.log('docs.json: valid')"
```

A single misplaced comma in `openapi.json` breaks the entire API reference section.

### Step 3 — OpenAPI integrity (light)

A full OpenAPI 3.x validator would be ideal; for now, spot-check the diff:

```bash
git diff main...HEAD -- api-reference/openapi.json | head -80
```

Look for:
- `$ref` paths that match component keys (typos = broken page)
- Missing required fields on schemas
- `deprecated: true` flag on legacy paths
- Tags consistent with the existing taxonomy

### Step 4 — Internal link check

A quick rg for broken relative links:

```!
rg "\]\(/" --type md --type mdx | head -40
```

For each link `[text](/foo)`, verify the corresponding file exists (`foo.mdx` or `foo/index.mdx`). Mintlify won't fail the build for broken internal links — it just renders 404s for customers.

### Step 5 — Mintlify dev preview (if available)

```bash
mintlify dev
```

Open `http://localhost:3000` (default), navigate to changed pages, verify rendering. Skip only if Mintlify CLI isn't installed locally.

### Step 6 — Cross-repo sync

Invoke `/cross-repo-sync` (sibling skill). For every change in the docs:

- Does the BE actually expose this endpoint?
- Does the SDK actually provide this method?
- Does the dashboard actually expose this feature?

Drift = customer confusion. If docs describe behavior that doesn't exist, file a bug — either fix the docs to match reality or fix reality to match docs.

### Step 7 — Branch + PR readiness

- Branch pushed: `git push -u origin <branch>`
- PR description includes: which BE/SDK/FE PRs this corresponds to, what changed, whether this can deploy independently or must coordinate

### Report

```
- ✅ JSON: openapi.json + docs.json valid
- ✅ Internal links: scanned, none broken
- ⚠️ Cross-repo: docs describe POST /connections/:id/rotate-credentials but BE PR #142 not yet merged → wait
- ✅ Mintlify dev: pages render
```
