# Cloudflare Pages Hosting Migration

**Date:** 2026-03-29
**Status:** Approved
**Supersedes:** `docs/specs/2026-03-23-cloudflare-hosting-design.md`
**Depends on:** RST → MyST Migration (must merge before DNS cutover)
**Related:** LLM Adaptation (parallel, URL update at cutover), Mobile Performance (Phase 3), Visual Editor (Phase 3)
**Execution:** Phase 2 — after MyST Migration merges. LLM Adaptation can proceed in parallel.

## Goal

Migrate VyOS documentation hosting from ReadTheDocs to Cloudflare Pages, gaining:

- Unlimited free bandwidth with 300+ edge PoP CDN
- Full control over cache headers (vs RTD's fixed 20-minute TTL)
- `_redirects` and `_headers` file support
- Cloudflare Workers + D1 as a future platform for the visual editor API
- No ads (RTD community tier injects EthicalAds)

## Platform Selection

Evaluated 7 hosting platforms. Cloudflare Pages won on every relevant criterion:

| | Cloudflare Pages | GitHub Pages | AWS S3+CF | Netlify | Vercel |
|---|---|---|---|---|---|
| Cost | $0 | $0 | ~$0.50/mo | $0 (20 deploys/mo) | $20/mo (commercial) |
| Bandwidth | Unlimited | 100 GB soft | 1 TB free | ~30 GB (credits) | 100 GB |
| CDN | 300+ PoPs | Fastly, no control | 600+ PoPs | Fair | Good |
| Cache control | `_headers` file | None | Full | `_headers` file | `vercel.json` |
| Redirects | `_redirects` (2100) | None | CF Functions | `_redirects` | 2048 routes |
| Future web app | Workers + D1 | No | Lambda/ECS | Functions | Functions |

GitHub Pages is the only credible alternative (free, simple) but lacks cache control, redirects, and serverless compute for the visual editor.

## Architecture

Unchanged from the original spec. Single CF Pages project, subdirectory-per-version:

```
Push to branch X
  └─ GHA: checkout → Docker build → optimize images → Pagefind → stage → CF Pages deploy
       └─ docs.vyos.io/en/{version}/   (HTML + downloads/)
```

## URL Structure

```
docs.vyos.io/en/1.5/     ← current (circinus, VyOS 1.5.x)
docs.vyos.io/en/1.4/     ← sagitta (VyOS 1.4.x)
docs.vyos.io/en/1.3/     ← equuleus (VyOS 1.3.x)
docs.vyos.io/en/1.2/     ← crux (VyOS 1.2.x)

docs.vyos.io/en/1.5/downloads/VyOS-1.5.pdf
docs.vyos.io/en/1.5/downloads/VyOS-1.5.epub
```

### Redirects (`_redirects` at repo root)

```
/                      /en/1.5/             302
/en/latest/*           /en/1.5/:splat       302
/en/stable/*           /en/1.4/:splat       302
/en/current/*          /en/1.5/:splat       302
/en/sagitta/*          /en/1.4/:splat       301
/en/equuleus/*         /en/1.3/:splat       301
/en/crux/*             /en/1.2/:splat       301
```

302 for aliases that change with releases; 301 for fixed historical branch names.

## Cache Headers (`_headers` at repo root)

```
# Versioned static assets — immutable, cache forever
/en/*/_static/*
  Cache-Control: public, max-age=31536000, immutable

# Pagefind search index — changes on rebuild
/en/*/pagefind/*
  Cache-Control: public, max-age=86400

# HTML pages — short browser cache, longer edge cache
/en/*/*.html
  Cache-Control: public, max-age=3600, s-maxage=86400

# PDF/EPUB downloads — stable per version
/en/*/downloads/*
  Cache-Control: public, max-age=604800

# LLM files — change on rebuild
/llms.txt
  Cache-Control: public, max-age=86400
/llms-full.txt
  Cache-Control: public, max-age=86400

# Sitemap — change on rebuild
/en/*/sitemap.xml
  Cache-Control: public, max-age=86400
```

This directly addresses the RTD performance limitation (20-minute cache TTL on all assets). Static assets get 1-year immutable cache. HTML gets 1-hour browser / 24-hour edge. Downloads get 7-day cache.

## GitHub Actions Workflow

**New file:** `.github/workflows/deploy-docs.yml`

```yaml
on:
  push:
    branches: [current, sagitta, equuleus, crux]
  workflow_dispatch:

concurrency:
  group: deploy-docs
  cancel-in-progress: false

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: recursive

      - name: Set version
        run: |
          case "${{ github.ref_name }}" in
            current)   echo "VERSION=1.5" >> $GITHUB_ENV ;;
            sagitta)   echo "VERSION=1.4" >> $GITHUB_ENV ;;
            equuleus)  echo "VERSION=1.3" >> $GITHUB_ENV ;;
            crux)      echo "VERSION=1.2" >> $GITHUB_ENV ;;
            *)         echo "ERROR: unknown branch ${{ github.ref_name }}" && exit 1 ;;
          esac

      - name: Restore other versions from cache
        uses: actions/cache/restore@v4
        with:
          path: dist
          key: vyos-docs-${{ github.run_id }}
          restore-keys: vyos-docs-

      - name: Authenticate to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_TOKEN }}

      - name: Build HTML
        run: |
          docker run --rm -v "$PWD":/vyos -w /vyos/docs \
            vyos/vyos-documentation make html

      - name: Optimize images
        run: |
          docker run --rm -v "$PWD":/vyos -w /vyos/docs \
            vyos/vyos-documentation make optimize-images

      - name: Build PDF
        run: |
          docker run --rm -v "$PWD":/vyos -w /vyos/docs \
            vyos/vyos-documentation make latexpdf

      - name: Build EPUB
        run: |
          docker run --rm -v "$PWD":/vyos -w /vyos/docs \
            vyos/vyos-documentation make epub

      - name: Stage output
        run: |
          mkdir -p dist/en/${{ env.VERSION }}/downloads
          cp -r docs/_build/html/. dist/en/${{ env.VERSION }}/
          cp docs/_build/latex/VyOS.pdf \
             dist/en/${{ env.VERSION }}/downloads/VyOS-${{ env.VERSION }}.pdf
          cp docs/_build/epub/VyOS.epub \
             dist/en/${{ env.VERSION }}/downloads/VyOS-${{ env.VERSION }}.epub
          if [ "${{ env.VERSION }}" = "1.5" ]; then
            cp _redirects dist/_redirects
            cp _headers dist/_headers
            cp docs/_build/html/404.html dist/404.html
          fi

      - name: Build Pagefind index
        run: |
          npx pagefind \
            --site dist/en/${{ env.VERSION }} \
            --output-path dist/en/${{ env.VERSION }}/pagefind

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CF_API_TOKEN }}
          accountId: ${{ secrets.CF_ACCOUNT_ID }}
          command: pages deploy dist --project-name=vyos-docs

      - name: Save merged dist to cache
        uses: actions/cache/save@v4
        with:
          path: dist
          key: vyos-docs-${{ github.run_id }}

      - name: Verify all versions live
        run: |
          for VERSION in 1.5 1.4 1.3 1.2; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
              https://vyos-docs.pages.dev/en/$VERSION/)
            if [ "$STATUS" != "200" ]; then
              echo "ERROR: /en/$VERSION/ returned $STATUS"
              exit 1
            fi
          done
```

### Differences from original spec workflow:
- **Added `make optimize-images`** step between HTML build and PDF build (mobile performance spec)
- **Added `_headers` copy** to stage step alongside `_redirects`

### Cache recovery workflow (`.github/workflows/rebuild-all.yml`)

Unchanged from original spec. Runs weekly (Monday 04:00 UTC) and on `workflow_dispatch`. Triggers all 4 branches in sequence.

### New secrets required

| Secret | Scope |
|---|---|
| `CF_API_TOKEN` | Cloudflare API token, Pages:Edit permission |
| `CF_ACCOUNT_ID` | Cloudflare account ID |
| `DOCKER_USER` | Docker Hub username |
| `DOCKER_TOKEN` | Docker Hub access token (read-only) |

## Search

**Pagefind** — fully static, zero cost, per-version index. JS snippet in `docs/_templates/layout.html` replaces the RTD search widget. Future option: Algolia DocSearch (free for OSS).

## Version Switcher

JS snippet in template. Reads active version from URL path, switches to same page in target version with HEAD-request fallback to version root.

## 404 Handling

Root-level `dist/404.html` copied from the `current` branch's `sphinx-notfound-page` output.

## Cross-Spec Interactions

### LLM Adaptation Spec
- `html_baseurl` must be updated to `https://docs.vyos.io/en/1.5/` at DNS cutover
- Curated `llms.txt` URLs must be updated from `/en/latest/` to `/en/1.5/`
- `llms.txt`, `llms-full.txt`, `robots.txt`, `sitemap.xml` are served automatically from HTML build output
- `sphinx-sitemap` generates per-version sitemap at `/en/{version}/sitemap.xml`
- `robots.txt` Sitemap directive should reference `https://docs.vyos.io/en/1.5/sitemap.xml`

### MyST Migration Spec
- No hosting impact — HTML output format is identical regardless of source format
- MyST migration should merge before DNS cutover to avoid two migrations on the live site

### Visual Editor Spec
- Editor frontend (static files) can deploy as a separate CF Pages project at `edit.docs.vyos.io`
- Staging API has two options:
  - **Cloudflare Workers + D1** — serverless, same platform, SQLite-compatible database. Consolidates infrastructure.
  - **Separate VPS** — as currently specified in the visual editor spec
- Decision deferred to visual editor implementation. This spec notes both options.

### Mobile Performance Spec
- `_headers` file enables cache header control (the primary RTD limitation)
- Image optimization (`make optimize-images`) integrated into GHA workflow
- CSS consolidation and JS deferring work independently of hosting platform but should be tested post-migration

## Migration Plan (zero-downtime)

### Step 0 — Bootstrap
Trigger `deploy-docs.yml` manually for each branch in order: `current` → `sagitta` → `equuleus` → `crux`. Seeds the cache chain.

### Step 1 — Parallel build
Deploy to staging domain (`vyos-docs.pages.dev`). RTD remains live.

### Step 2 — Validate on staging
- All 4 versioned paths render correctly
- Version switcher works
- Search returns scoped results per version
- PDF and EPUB downloads work
- `_redirects` route legacy URLs correctly
- `_headers` cache rules apply correctly
- 404 pages display correctly
- `llms.txt` and `llms-full.txt` accessible at root
- `sitemap.xml` generated per version

### Step 3 — DNS cutover
- Point `docs.vyos.io` CNAME to CF Pages
- Update `html_baseurl` to `https://docs.vyos.io/en/1.5/`
- Update curated `llms.txt` URLs from `/en/latest/` to `/en/1.5/`
- Update `robots.txt` Sitemap directive to `https://docs.vyos.io/en/1.5/sitemap.xml`
- Submit sitemap to search engines

### Step 4 — Decommission RTD
Remove `.readthedocs.yml`, disable RTD project. Retain RTD account 30 days as rollback.

## Execution Order (All Five Specs)

| Phase | Specs | Can start |
|---|---|---|
| Phase 1 (parallel) | LLM Adaptation + MyST Migration | Immediately |
| Phase 2 | Cloudflare Hosting Migration | After MyST Migration merges |
| Phase 3 (parallel) | Mobile Performance + Visual Editor | After Cloudflare Migration completes |

## Risks

- **Cache chain eviction** — GitHub caches expire after 7 days of inactivity. Mitigation: weekly `rebuild-all.yml` workflow.
- **Docker Hub rate limits** — authenticated pulls required. Mitigation: `DOCKER_USER`/`DOCKER_TOKEN` secrets.
- **DNS propagation** — some users may see RTD during propagation. Mitigation: both platforms serve valid content; RTD stays live until propagation completes.
- **File limit** — CF Pages free tier allows 20,000 files per deployment. 4 versions × ~2,000 files = ~8,000 files. Well within limits.

## Maintenance

- **GHA workflow**: Update when new VyOS release branches are created (add to branch list, version mapping, and VERSIONS array).
- **`_redirects`**: Update when `/en/latest/` should point to a new version.
- **`_headers`**: Rarely changes.
- **Secrets**: Rotate CF API token and Docker Hub token per their policies.
