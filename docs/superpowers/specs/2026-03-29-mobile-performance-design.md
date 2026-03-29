# Mobile Performance & Core Web Vitals Optimization

**Date:** 2026-03-29
**Status:** Approved
**Depends on:** Cloudflare Hosting Migration (cache headers require CF Pages; image optimization integrated into GHA workflow)
**Related:** LLM Adaptation spec (independent), Visual Editor spec (independent), RST → MyST Migration (safer to optimize post-migration codebase)
**Execution:** Phase 3 — after Cloudflare Migration completes. Can run in parallel with Visual Editor.

## Goal

Optimize the VyOS documentation site for mobile performance and Core Web Vitals (LCP, INP, CLS) to improve:

1. Mobile user experience (reported complaints about slow loading)
2. Google search rankings via Core Web Vitals signals
3. Overall page load performance for all users

## Current State (Audit Findings)

The live site at docs.vyos.io has significant performance issues:

| Issue | Impact | Size |
|---|---|---|
| Header iframe loading a full Next.js app | ~25 extra requests, layout shift | ~120 KB HTML + assets |
| `datatables.js` loaded synchronously on every page | Blocks rendering, most pages don't use it | 454 KB |
| 20 render-blocking CSS files (15 custom) | Waterfall of blocking requests | ~192 KB total |
| 13 synchronous JS files (none deferred) | Blocks interactivity | ~560 KB |
| 37 MB of unoptimized images (190 files) | No WebP, no lazy loading, no dimensions | Up to 2.6 MB per image |
| Google Fonts Archivo as TTF (not WOFF2) | Render-blocking external fetch | 5 font weights |
| FontAwesome 4.7 without `font-display: swap` | Flash of invisible text | 77 KB WOFF2 |
| 20-minute cache TTL on all assets | RTD platform limitation (not controllable) | — |

## Approach

Template overrides for HTML changes (script defer, preload hints, iframe optimization). External tooling (Pillow) as a post-build step for image optimization. CSS consolidation done manually once.

## Design

### 1. JavaScript Optimization

**Defer all custom scripts** via template override in `docs/_templates/layout.html`. jQuery remains synchronous (sphinx_rtd_theme hard dependency). Everything else gets `defer`.

| Script | Loading | Rationale |
|---|---|---|
| `jquery.js` | Synchronous | Theme hard dependency |
| `datatables.js` | `defer` | Only needs DOM-ready, not render-blocking |
| All other custom JS (`doctools.js`, `sphinx_highlight.js`, `design-tabs.js`, `theme.js`, `tables.js`, `codecopier.js`, `sidebar.js`, `footer.js`, etc.) | `defer` | None require synchronous execution |
| `readthedocs-addons.js` | Already `async` | No change needed |

### 2. CSS Consolidation

**Merge all 15 custom CSS files into a single `vyos.css`.** One-time manual task — concatenate in dependency order, verify no selector conflicts.

Files to merge:
- `custom.css`, `lists.css`, `hints.css`, `headers.css`, `breadcrumbs.css`, `linkButtons.css`, `text.css`, `leftSidebar.css`, `scrolls.css`, `tables.css`, `running-on-bare-metal.css`, `code-snippets.css`, `separate-commands.css`, `configuration/index.css`, `datatables.css`

Update `conf.py` — replace 15 entries in `html_css_files` with the single `vyos.css`.

Theme-managed CSS files stay separate (`theme.css`, `pygments.css`, `design-style.*.min.css`, `graphviz.css`).

**Result:** 20 CSS files → 6 CSS files. Eliminates 14 render-blocking requests.

### 3. Font Optimization

**Self-host Archivo as WOFF2:**
- Download WOFF2 files for 5 weights (400, 500, 600, 700, 800)
- Place in `docs/_static/fonts/`
- Add `@font-face` declarations to `vyos.css` with `font-display: swap`
- Remove the Google Fonts `<link>` from the template

Benefits: eliminates external DNS lookup + TLS handshake to `fonts.googleapis.com` and `fonts.gstatic.com`, serves WOFF2 instead of TTF (~30-50% smaller), removes render-blocking external request.

**Add `font-display: swap` to FontAwesome** via CSS override:
```css
@font-face {
  font-family: 'FontAwesome';
  font-display: swap;
  /* re-declare src from theme.css */
}
```

Dead Lato/Roboto Slab references in theme CSS are harmless (not loaded externally) — no action needed.

### 4. Header Iframe Optimization

The header iframe (`<iframe src='https://vyos.io/iframes/header'>`) stays but gets three optimizations:

1. **Set initial height** — hardcode `height` attribute matching the header's rendered height to prevent layout shift (CLS).
2. **Add `loading="lazy"`** — signals non-critical resource to the browser.
3. **Add `preconnect` hint** — `<link rel="preconnect" href="https://vyos.io">` to reduce connection setup delay. Verify if already present in template.

### 5. Image Optimization

A post-build Python script using `Pillow` for image operations and `BeautifulSoup` for HTML rewriting.

**What it does:**

1. **Convert to WebP** — convert all PNG/JPG in `_build/html/_static/images/` to WebP via Pillow. Keep originals as fallback.

2. **Resize oversized images** — cap maximum width at 1200px. Images above this threshold are downscaled.

3. **Generate `<picture>` elements** — rewrite `<img>` tags in built HTML:
   ```html
   <picture>
     <source srcset="image.webp" type="image/webp">
     <img src="image.png" ...>
   </picture>
   ```

4. **Add `loading="lazy"` to all images** — except the first image per page (above-the-fold, should load eagerly).

5. **Add `width` and `height` attributes** — read actual image dimensions with Pillow, inject into `<img>` tags to prevent CLS.

**Integration:** New `make optimize-images` Makefile target that runs after `make html`. Docker build chains them: `make html && make optimize-images`.

**Dependencies:** `Pillow` and `beautifulsoup4` added to `requirements.txt`. Both are pure Python — no system-level dependencies for WebP support (Pillow includes WebP support by default).

**Docker image:** Adding `Pillow` and `beautifulsoup4` to `requirements.txt` is sufficient if the Docker image installs from `requirements.txt` at build time. Verify at implementation time whether the `vyos/vyos-documentation` image needs rebuilding.

## Risks

- **Template override fragility** — custom `layout.html` must track upstream `sphinx_rtd_theme` changes on theme upgrades. Mitigation: the override is minimal (adding attributes to existing tags, not restructuring).
- **CSS consolidation regressions** — merging 15 files could surface selector specificity issues. Mitigation: visual diff the built site before and after.
- **Image optimization build time** — processing 190 images adds to build time. Mitigation: the script can skip already-converted images (check file timestamps).
- **WebP browser support** — WebP is supported by all modern browsers. The `<picture>` fallback handles older browsers.

## Maintenance

- **`vyos.css`**: Ongoing — new custom styles go here instead of creating new CSS files.
- **Font files**: Static — only update if Archivo font family is changed.
- **Image optimization script**: Minimal — runs automatically as part of build.
- **Template overrides**: Check on `sphinx_rtd_theme` upgrades.

## Expected Impact

| Metric | Before | After (estimated) |
|---|---|---|
| Render-blocking CSS requests | 20 | 6 |
| Render-blocking JS | 13 files, ~560 KB | 1 file (jQuery, ~89 KB) |
| External font requests | 5 TTF files from Google Fonts | 0 (self-hosted WOFF2) |
| Image payload (typical page) | Unoptimized PNG/JPG | WebP, lazy-loaded, correctly sized |
| CLS from header iframe | Significant | Eliminated (hardcoded height) |
| CLS from images | Significant | Eliminated (width/height attributes) |
