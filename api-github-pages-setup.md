# Hosting API docs on GitHub Pages

This guide explains how to publish the Swagger UI for PhotoVault's API at a public URL like:

```
https://jarkendar.github.io/PhotoVault/api/
```

The setup uses GitHub Pages with a static site served from the `docs/` folder. No build step, no GitHub Actions required.

## Prerequisites

- Files committed to your repository:
  - `docs/api.md`
  - `docs/api-cheatsheet.md`
  - `docs/api/openapi.yaml`
  - `docs/api/index.html`

- Repository is public (GitHub Pages on private repos requires a paid plan).

## Setup steps

### 1. Commit and push the docs

```bash
git add docs/
git commit -m "docs: add API contract (Markdown + OpenAPI + Swagger UI)"
git push
```

### 2. Enable GitHub Pages

1. Open the repository in browser: `https://github.com/Jarkendar/PhotoVault`
2. Go to **Settings** (top-right of the repo)
3. In the left sidebar: **Pages**
4. Under "Build and deployment":
   - **Source**: Deploy from a branch
   - **Branch**: `master` (or `main` if you renamed it) + folder `/docs`
5. Click **Save**

GitHub will start the deployment. Within 1-2 minutes the page is live.

### 3. Verify

Open in browser:

```
https://jarkendar.github.io/PhotoVault/api/
```

You should see the Swagger UI rendering all endpoints from `openapi.yaml`. If you see a 404, wait another minute (first deployment can take a few minutes), then refresh.

To check deployment status: **Settings → Pages** shows the latest build. Or: **Actions** tab shows the `pages-build-deployment` workflow.

## Updating after API changes

The Swagger UI is automatically updated whenever `openapi.yaml` changes — there is no separate publish step. Workflow:

1. Edit `docs/api/openapi.yaml`
2. (Optionally) update `docs/api.md` and `docs/api-cheatsheet.md` to keep narrative in sync
3. Commit and push
4. GitHub Pages rebuilds automatically (~1 minute)

## Adding a link in the main README

Once the page is live, add a link in the project's main `README.md`:

```markdown
## API documentation

- **Narrative:** [docs/api.md](docs/api.md)
- **Quick reference (PL):** [docs/api-cheatsheet.md](docs/api-cheatsheet.md)
- **Interactive (Swagger UI):** [https://jarkendar.github.io/PhotoVault/api/](https://jarkendar.github.io/PhotoVault/api/)
- **OpenAPI spec:** [docs/api/openapi.yaml](docs/api/openapi.yaml)
```

## Troubleshooting

### Swagger UI shows "Failed to load API definition"

The `openapi.yaml` path in `index.html` is relative. Check that the file is exactly at `docs/api/openapi.yaml` and that `docs/api/index.html` references it as `url: 'openapi.yaml'` (without leading slash).

### 404 when visiting `/api/`

GitHub Pages serves `index.html` from a directory by default. Confirm `docs/api/index.html` exists and is committed. If it does and you still see 404, check that GitHub Pages source is set to `/docs` folder, not the root.

### YAML validation errors

Use a local validator before committing:

```bash
npx @redocly/cli lint docs/api/openapi.yaml
```

Or paste the YAML into [editor.swagger.io](https://editor.swagger.io/) — it parses on the fly.

### "Try it out" button is grayed out

In `index.html`, `tryItOutEnabled` is set to `false` to avoid CORS errors when the server isn't running. To enable it:

1. Set `tryItOutEnabled: true` in `index.html`
2. Configure CORS on the Ktor server to allow the GitHub Pages origin (or `*` during development)

## Optional: custom domain

If you own a domain (e.g., `photovault.dev`):

1. Add a `CNAME` file in `docs/` containing only the domain: `photovault.dev`
2. Configure DNS: `CNAME photovault.dev → jarkendar.github.io`
3. In Settings → Pages, enter the custom domain and enable HTTPS

Then the docs are at `https://photovault.dev/api/`.

## Optional: OpenAPI lint as part of CI

To prevent broken specs from reaching the published site, add a GitHub Action:

```yaml
# .github/workflows/openapi-lint.yml
name: OpenAPI lint
on:
  pull_request:
    paths: ['docs/api/openapi.yaml']
  push:
    branches: [master]
    paths: ['docs/api/openapi.yaml']

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npx @redocly/cli@latest lint docs/api/openapi.yaml
```

This is optional but cheap insurance — it catches typos before they reach production.
