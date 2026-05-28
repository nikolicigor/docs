# FileFeed Docs — Engineering Rules

Mintlify-powered public docs. **What's here is the customer's contract.** A wrong line in `openapi.json` or a stale code example is a real bug — customers integrate against this site.

## What this repo is

- Mintlify static site, deployed on push to `main`
- `docs.json` — navigation + theme
- `api-reference/openapi.json` — the OpenAPI 3.x spec; source of truth for API shape
- `*.mdx` — content pages with React-flavored markdown (`<CodeGroup>`, `<Tip>`, `<Note>`, `<Steps>`)
- `migration/*.mdx` — per-API-version upgrade guides
- `changelog.mdx` — customer-facing changelog (API-perspective, not internal-engineering)

Sibling repos that MUST stay in sync with this one:
- Backend: `/Users/Nikolic-Marko/FileFeed/filefeed-backend` — owns the actual API
- SDK: `/Users/Nikolic-Marko/FileFeed/filefeed-backend/filefeed-sdk` — owns SDK examples
- Frontend: `/Users/Nikolic-Marko/FileFeed/filefeed-frontend` — owns dashboard screenshots

## The non-negotiables

1. **`api-reference/openapi.json` must be valid JSON and match the BE.** Run `node -e "JSON.parse(require('fs').readFileSync('api-reference/openapi.json','utf8'))"` before commit. Spec ↔ BE drift is the worst kind of docs bug — customers integrate, customers fail, customers churn.
2. **No direct commits to `main`.** Branch → PR → merge. Same as the other repos.
3. **Every BE API change has a docs PR — preferably the same PR.** If you can't ship them together, the BE PR description must link the planned docs PR and the docs PR must merge before the BE feature flag flips to on.
4. **Code examples must run.** Curl examples must use real endpoint shapes; SDK examples must compile against the current published SDK version. If you're updating examples for an unreleased SDK version, mark the page with `<Note>` callouts.
5. **No marketing language in technical pages.** "Easily" / "seamlessly" / "blazing-fast" → cut. Show the code; the work speaks.

## File conventions

- File names: kebab-case `.mdx`. Example: `migration/v1-to-v2.mdx`.
- Frontmatter at the top of every `.mdx`:
  ```mdx
  ---
  title: "Connections"
  description: "Manage SFTP / API endpoints that pipelines ingest from."
  ---
  ```
- Headings: `#` is the page H1 (rendered from `title` in frontmatter — don't repeat). Start content at `##`.
- Links: relative paths (`/migration/v1-to-v2`) not absolute (`https://docs.filefeed.io/...`).
- Cross-link liberally — every concept should be reachable in two clicks from the docs home.

## API reference rules

### `openapi.json`

- One source of truth. Don't duplicate schema definitions across pages.
- Every endpoint has: summary, description, parameters, request body schema, response schema (per status code), tags
- Deprecated endpoints: `deprecated: true` + tag suffix `(deprecated)` (e.g. `Clients (deprecated)`)
- Schema definitions live under `components.schemas`. Reuse via `$ref`.
- Rename aliases: type aliases as `allOf` with the deprecated copy referencing the new one

### Per-resource `.mdx` pages

- Page title = resource (singular noun): "Connection", "Pipeline", "Webhook"
- Sections: Overview → Object shape → Endpoints (one section per HTTP verb) → Examples
- Examples: `<CodeGroup>` with SDK + curl + (optional) TypeScript types, in that order

## Migration guides

When a new API version ships:

1. Create `migration/<source>-to-<target>.mdx` (or update if exists)
2. Sections: Overview, What changed (numbered), Before/after for each change, Action required
3. Each "before/after" is a `<CodeGroup>` with SDK + curl
4. Link from `changelog.mdx` and from the relevant resource page
5. Update `docs.json` navigation to include the new migration page in the "Versioning" group

## Changelog

`changelog.mdx` is **customer-facing**. API-first language:

```mdx
## 2026-05-25

### Added
- New endpoint: `POST /connections/:id/rotate-credentials` …

### Deprecated
- `/clients/*` legacy alias. Use `/connections/*`. Sunset 2027-05-25.
  See [Migration v1 → v2](/migration/v1-to-v2).

### Sunset
- (nothing yet)
```

Order: most recent at the top. Each section dated with the API version it shipped under.

The SDK has its own changelog at `filefeed-sdk/CHANGELOG.md` — different audience (npm consumers), different perspective (SDK methods). Don't try to consolidate.

## Code examples policy

- Default to **TypeScript SDK** examples on every endpoint page
- Include **curl** for raw-HTTP integrators
- Skip language examples we don't officially support (Python, Ruby, Go) — adding partial samples implies SDK availability we don't have
- All examples include the `FileFeed-Version` header in curl examples — customers seeing the header in docs learn to set it

### Example shape

```mdx
<CodeGroup>

```ts SDK
const filefeed = new FileFeed({ apiKey: process.env.FILEFEED_API_KEY });
const conn = await filefeed.connections.create({
  name: "Acme :: Production",
  useHostedSFTP: true,
  awsPassword: "<strong-password>",
});
```

```bash curl
curl https://api.sftpsync.io/connections \
  -X POST \
  -H "x-api-key: $FILEFEED_API_KEY" \
  -H "FileFeed-Version: 2026-05-25" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme :: Production",
    "useHostedSFTP": true,
    "awsPassword": "<strong-password>"
  }'
```

</CodeGroup>
```

## Branch + commit + cross-repo

Same as backend / frontend:

- Branch names: `feat/*`, `fix/*`, `chore/*`, `docs/*`
- Conventional commits in messages
- One logical change per PR
- Never push directly to `main`

## Verification

Before merge:

```bash
# Validate openapi.json
node -e "JSON.parse(require('fs').readFileSync('api-reference/openapi.json','utf8')); console.log('openapi.json: valid')"

# Validate docs.json
node -e "JSON.parse(require('fs').readFileSync('docs.json','utf8')); console.log('docs.json: valid')"

# Mintlify CLI build check (if installed)
mintlify dev  # local preview
```

If Mintlify CLI isn't installed, the production build catches errors on deploy — but it's better to catch them locally.

## When in doubt

- The `automated-flows/` and `api-reference/` pages are the canonical structure examples — match their format
- Look at the existing `migration/v1-to-v2.mdx` for the migration guide template
- Check `changelog.mdx` for the changelog entry shape
- The OpenAPI spec is the source of truth; if the prose page disagrees with the spec, the prose is wrong
