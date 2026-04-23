---
title: "VyOS Documentation Modernization"
subtitle: "Project Summary — Six Initiatives"
date: "2026-03-29"
---

# VyOS Documentation Modernization

## Executive Summary

This document summarizes six coordinated initiatives to modernize the VyOS documentation platform. The project addresses documentation contributor accessibility, LLM readiness, hosting performance, visitor analytics, and mobile experience. All initiatives have approved design specs and a defined execution order.

**Current state:** 258 reStructuredText files built with Sphinx, hosted on ReadTheDocs at docs.vyos.io. Custom extensions for VyOS CLI command documentation. Docker-based build pipeline.

**Target state:** MyST Markdown source, hosted on Cloudflare Pages with full CDN control, visual editor for non-technical contributors, LLM-optimized output, server-side analytics, and optimized mobile performance.

---

## The Six Initiatives

### 1. LLM Documentation Adaptation

**Problem:** LLMs cannot reliably answer VyOS questions because the documentation lacks structured machine-readable output.

**Solution:** Add `sphinx-llms-txt` extension to auto-generate `llms-full.txt` (entire docs as markdown). Hand-maintain a curated `llms.txt` entry point with VyOS-specific context (set/delete/commit model, CLI hierarchy). Update `robots.txt` to explicitly allow AI crawlers. Add `sphinx-sitemap` for search engine and AI crawler discoverability.

**Effort:** Low. Two new Sphinx extensions, two static files, conf.py changes.

**Outcome:** VyOS documentation consumable by ChatGPT, Claude, Perplexity, and AI search products. Matches industry practice (Stripe, Vercel, Cloudflare).

---

### 2. RST to MyST Markdown Migration

**Problem:** reStructuredText is a barrier for contributors. No visual editor or CMS supports RST. Markdown is far more widely known.

**Solution:** Batch-convert all 258 RST files to MyST Markdown using `rst-to-myst` with custom VyOS directive mappings. The project already has `myst_parser` enabled and a `cmdincludemd` directive prepared for this migration. Add "Edit this page" links to every documentation page, enabling drive-by contributions via GitHub's web editor.

**Effort:** Medium. One-time batch conversion with 20-40% of files requiring manual review. Validated by HTML diff before/after.

**Outcome:** Contributors write Markdown instead of RST. "Edit this page" links enable contributions without local Git setup. Unlocks the visual editor (Initiative 4).

---

### 3. Cloudflare Hosting Migration

**Problem:** ReadTheDocs has a fixed 20-minute cache TTL, no cache header control, injects ads on the free tier, and cannot run custom Docker builds natively.

**Solution:** Migrate to Cloudflare Pages. GitHub Actions builds each branch using the existing Docker image, deploys to a single CF Pages project with subdirectory-per-version (`/en/1.5/`, `/en/1.4/`, etc.). Pagefind replaces RTD's built-in search. A custom version switcher replaces RTD's flyout. `_headers` file enables aggressive caching for static assets.

**Effort:** Medium. New GHA workflow, Pagefind integration, version switcher JS, redirect rules, zero-downtime DNS cutover.

**Outcome:** Unlimited free bandwidth, 300+ edge PoP CDN, full cache control, no ads, platform for CF Workers (analytics, future visual editor API).

**Evaluated alternatives:** GitHub Pages (viable but no cache control/redirects), AWS S3+CloudFront (maximum control but high maintenance), Netlify (restrictive credit-based pricing), Vercel (commercial use restriction on free tier).

---

### 4. Visual Documentation Editor

**Problem:** Non-technical contributors (network engineers, docs team members) cannot contribute because the workflow requires Git knowledge.

**Solution:** Deploy the Antmicro MyST Editor (open-source, Preact-based) as a web application with live MyST preview including VyOS-specific directive rendering. Edits save to a staging database (SQLite). Maintainers review pending edits via an admin interface and approve/reject before changes reach Git. Authentication via Auth0 (existing tenant) with contributor and maintainer roles.

**Effort:** High. New web application: editor frontend (static), staging API (lightweight HTTP service), Auth0 integration, Git operations via GitHub App token.

**Outcome:** Non-technical contributors can edit documentation with live preview, no Git knowledge required. All edits go through maintainer review. "Edit this page" links (from Initiative 2) serve drive-by contributors; the visual editor serves sustained content work.

---

### 5. Mobile Performance & Core Web Vitals

**Problem:** The docs site has significant performance issues: 20 render-blocking CSS files, 13 synchronous JS files (454 KB DataTables on every page), unoptimized images (37 MB, no WebP, no lazy loading), render-blocking Google Fonts, and a header iframe loading a full Next.js app.

**Solution:**

- **JS:** Defer all custom scripts (jQuery stays synchronous as theme dependency). DataTables no longer blocks rendering.
- **CSS:** Consolidate 15 custom CSS files into a single `vyos.css`. Reduces render-blocking requests from 20 to 6.
- **Fonts:** Self-host Archivo as WOFF2 (replaces Google Fonts TTF). Add `font-display: swap` to FontAwesome.
- **Images:** Post-build Python script converts to WebP, resizes oversized images, adds `loading="lazy"` and `width`/`height` attributes, generates `<picture>` elements with fallback.
- **Header iframe:** Set initial height (prevents layout shift), add `loading="lazy"`, add preconnect hint.

**Effort:** Medium. Template overrides, one-time CSS consolidation, post-build image optimization script (Pillow + BeautifulSoup).

**Outcome:** Dramatically improved Core Web Vitals (LCP, INP, CLS). Fewer render-blocking resources, optimized images, controlled caching via CF Pages `_headers` file.

---

### 6. Visitor Tracking & Analytics

**Problem:** No analytics on docs.vyos.io. No visibility into which documentation is used, how visitors navigate, or whether docs drive product conversions.

**Solution:** Two-layer architecture:

- **Layer 1 (consented):** Extend the existing GTM + CookieBot + sGTM stack (already running on metrics.vyos.io for vyos.io, blog, and forum) to docs.vyos.io. Add HubSpot tracking via sGTM. Cross-domain measurement with vyos.io for conversion attribution.
- **Layer 2 (cookieless):** CF Worker writes every HTML page request to Cloudflare Analytics Engine. True server-side, no cookies, no consent needed. Captures pageviews, referrers, countries, doc versions for the 30-50% of EU visitors who reject cookies.
- **Data warehouse:** GA4 native export to BigQuery (daily). Scheduled CF Worker exports Analytics Engine data to BigQuery. Unified reporting via Looker Studio or Grafana.

**Effort:** Low-Medium. Layer 1 is template snippets + GTM configuration. Layer 2 is ~100-200 lines of Worker code. BigQuery integration is configuration + a scheduled Worker.

**Outcome:** Full marketing analytics on docs. Consented users get rich GA4/HubSpot tracking. All users produce cookieless baseline data. All data in BigQuery for long-term analysis. Future-ready for IP-to-company enrichment.

---

## Execution Plan

### Phase 1 — Immediate (parallel)

| Initiative | Scope |
|---|---|
| LLM Documentation Adaptation | Full implementation |
| RST to MyST Migration + Edit Links | Full implementation |
| Visitor Tracking (Phase A) | GTM + CookieBot snippets in Sphinx templates |

No dependencies between these three. Can start immediately and run concurrently. Minor conf.py merge conflict between LLM and MyST specs (extensions list) — resolve when second branch merges.

### Phase 2 — After MyST Migration Merges

| Initiative | Scope |
|---|---|
| Cloudflare Hosting Migration | Full implementation including DNS cutover |

Depends on MyST Migration being complete to avoid migrating source format on the new platform. LLM Adaptation URL updates (`html_baseurl`, `llms.txt` URLs) happen at DNS cutover.

### Phase 3 — After Cloudflare Migration (parallel)

| Initiative | Scope |
|---|---|
| Mobile Performance | Full implementation |
| Visual Documentation Editor | Full implementation |
| Visitor Tracking (Phases B-D) | CF Worker, Analytics Engine, BigQuery |

All three require Cloudflare Pages to be live. Can run concurrently as they have no shared dependencies.

---

## Infrastructure Summary

### What Changes

| Component | Before | After |
|---|---|---|
| Source format | reStructuredText | MyST Markdown |
| Hosting | ReadTheDocs | Cloudflare Pages |
| Build | RTD builds | GitHub Actions + Docker |
| Search | RTD built-in (Elasticsearch) | Pagefind (static, client-side) |
| Version switcher | RTD flyout | Custom JS in template |
| Cache control | Fixed 20-min TTL | Custom `_headers` (1-year for static assets) |
| Analytics | None | GTM/sGTM + CF Worker + BigQuery |
| LLM access | None | llms.txt + llms-full.txt + sitemap |
| Contributor editing | Git + RST only | Git + Markdown + visual editor + "Edit this page" links |

### New Infrastructure Required

| Component | Purpose | Cost |
|---|---|---|
| Cloudflare Pages (free) | Documentation hosting | $0 |
| Cloudflare Workers (free tier) | Analytics Worker, BigQuery export | $0 |
| Cloudflare Analytics Engine (free tier) | Cookieless pageview storage | $0 |
| BigQuery (free tier) | Data warehouse | $0 (1TB query/mo, 10GB storage) |
| Visual editor API server | Staging API + SQLite | ~$10-30/mo VPS (or CF Workers + D1) |
| Grafana Cloud (free tier) | Analytics Engine dashboard | $0 (up to 3 users) |

### Existing Infrastructure Reused

| Component | Already serves |
|---|---|
| sGTM at metrics.vyos.io | vyos.io, blog.vyos.io, forum.vyos.io |
| CookieBot | All vyos.io properties |
| GTM container | All vyos.io properties |
| GA4 | All vyos.io properties |
| Auth0 tenant | Existing user authentication |
| Docker image (vyos/vyos-documentation) | Current documentation builds |

### New Secrets Required

| Secret | Scope |
|---|---|
| `CF_API_TOKEN` | Cloudflare Pages deploy |
| `CF_ACCOUNT_ID` | Cloudflare account |
| `DOCKER_USER` / `DOCKER_TOKEN` | Docker Hub authenticated pulls |

---

## Design Specs Reference

All detailed design specifications are located in the repository at `docs/superpowers/specs/`:

1. `2026-03-29-llm-documentation-adaptation-design.md`
2. `2026-03-29-rst-to-myst-migration-design.md`
3. `2026-03-29-cloudflare-hosting-migration-design.md`
4. `2026-03-29-visual-editor-design.md`
5. `2026-03-29-mobile-performance-design.md`
6. `2026-03-29-visitor-tracking-design.md`

Superseded: `docs/specs/2026-03-23-cloudflare-hosting-design.md` (replaced by spec #3 above).
