# photovault-api-contract

Single source of truth for the **PhotoVault REST API** (`/v1`). This repository is consumed
as a **git submodule** by both sides of the system so they pin the exact same contract
version:

- **Android client** — mounted at `docs/api/`
- **Ktor server** — mounted at `contract/`

This decouples the contract from either codebase: a change here is reviewed once, and each
side bumps the submodule SHA when ready to adopt it.

## Files

| File | Purpose |
|------|---------|
| `openapi.yaml` | Machine-readable OpenAPI 3.1 spec. **Authoritative — wins on any conflict.** |
| `api.md` | Narrative documentation and design rationale. |
| `api-cheatsheet.md` | Condensed quick reference (PL). |
| `index.html` | Swagger UI page for GitHub Pages (renders `openapi.yaml`). |

## Viewing the docs

Enable **GitHub Pages** on this repo (Settings → Pages → deploy from `main`, root). The
spec is then browsable at `https://<user>.github.io/photovault-api-contract/`.

Locally:

```bash
python3 -m http.server 8000   # then open http://localhost:8000
```

## Consuming as a submodule

Add it once on each side:

```bash
# Android client (replaces the old docs/api directory, same path)
git submodule add <this-repo-url> docs/api

# Ktor server
git submodule add <this-repo-url> contract
```

Clone a consumer with submodules in one shot:

```bash
git clone --recurse-submodules <consumer-repo-url>
# or, after a plain clone:
git submodule update --init --recursive
```

Bump to the latest contract later:

```bash
cd docs/api          # or cd contract on the server
git pull origin main
cd -
git add docs/api     # stage the new submodule SHA
git commit -m "build: bump API contract"
```

## Versioning

The API itself is versioned by URL prefix (`/v1`). Forward-compatible changes (new fields,
new endpoints, new optional params) stay in `v1`; breaking changes introduce `/v2`. See the
*Versioning* section of `api.md`.

## License

TBD (matches the `license: TBD` placeholder in `openapi.yaml`).
