# Visitor Tracking & Analytics

**Date:** 2026-03-29
**Status:** Approved
**Depends on:** Cloudflare Hosting Migration (for CF Worker + Analytics Engine; GTM snippets can be added immediately)
**Related:** All other specs (independent, no shared infrastructure)
**Execution:** Phase A (GTM snippets) starts immediately in Phase 1. Phases B-D start in Phase 3 after Cloudflare Migration.

## Goal

Add visitor tracking to docs.vyos.io for marketing purposes:

1. **Page-level traffic** — which pages are popular, traffic trends, referral sources
2. **User journey** — how visitors navigate through docs, entry/exit pages
3. **Conversion tracking** — visitors who go from docs to vyos.io (product pages, subscriptions)
4. **GDPR compliance** — cookie consent with CookieBot, cookieless fallback for non-consented users
5. **Data warehouse** — all analytics data in BigQuery for long-term retention and custom analysis

Future option: company/org identification from IP for sales intelligence.

## Existing Infrastructure

The VyOS organization already operates:

- **CookieBot** — consent management across vyos.io properties
- **GTM (client-side)** — tag management
- **Server-side GTM (sGTM)** — running at `metrics.vyos.io`, serving vyos.io, blog.vyos.io, forum.vyos.io
- **GA4** — analytics via sGTM

docs.vyos.io becomes another property in this existing stack. No new consent management or tag delivery infrastructure needed.

## Architecture

Two layers:

```
Consented users:
  Browser → CookieBot consent → GTM → sGTM (metrics.vyos.io) → GA4 → BigQuery

All users (cookieless):
  Request → CF Worker → Analytics Engine → BigQuery (scheduled export)
                ↓
         CF Pages (response)
```

**Layer 1 (GTM + CookieBot):** Rich tracking with consent — GA4 pageviews, events, HubSpot contact tracking, cross-domain measurement with vyos.io. Requires cookie consent.

**Layer 2 (CF Worker + Analytics Engine):** Cookieless server-side baseline — aggregate pageviews, referrers, countries, version popularity. No consent needed. Provides visibility into the 30-50% of EU visitors who reject cookies.

## Layer 1: GTM + CookieBot + HubSpot Integration

### Sphinx Template Changes

Add to `docs/_templates/layout.html`:

1. **CookieBot script** in `<head>` (before any tracking scripts) — uses the same CBID already deployed on vyos.io properties
2. **GTM container snippet** in `<head>` (after CookieBot) — same container ID as other vyos.io properties
3. **GTM noscript fallback** after opening `<body>` tag

### GTM Configuration Changes (in GTM UI, not code)

- Add `docs.vyos.io` as an allowed domain in the GTM container settings
- Add HubSpot tracking tag to sGTM if not already configured
- Configure GA4 cross-domain measurement to include `docs.vyos.io` (enables user journey tracking across vyos.io ↔ docs.vyos.io)

### What This Provides

- Page views, sessions, user journeys in GA4
- HubSpot contact tracking and page analytics (for identified contacts)
- Cross-domain conversion attribution (docs → vyos.io product pages)
- Referral source tracking
- All data flows through existing sGTM at metrics.vyos.io

### No New Infrastructure

sGTM is already running. CookieBot is already configured. GA4 is already collecting. This is purely template changes + GTM tag configuration.

## Layer 2: CF Worker — Cookieless Analytics

A Cloudflare Worker sits in front of CF Pages and writes a data point to Analytics Engine on every HTML page request.

### What It Captures

| Data | Source | Example |
|---|---|---|
| URL path | Request URL | `/en/1.5/configuration/firewall/zone.html` |
| Referrer | `Referer` header | `https://google.com/` |
| Country | `cf.country` property | `DE` |
| Browser family | Parsed from User-Agent | `Chrome` |
| OS family | Parsed from User-Agent | `Linux` |
| Device type | Parsed from User-Agent | `mobile` / `desktop` / `tablet` |
| Doc version | Extracted from URL path | `1.5` |

### What It Does NOT Capture

- IP address — not stored, not logged
- Raw User-Agent — hashed after parsing, not stored
- No cookies, no client identifiers
- No session tracking (each request is independent)

### Filtering

- Only track HTML page requests (path ends in `.html` or `/`)
- Skip static assets, images, fonts, Pagefind index, PDFs
- Skip bot-like User-Agents (basic filter)

### Worker Logic

```
Request arrives
  → Is it an HTML page?
    → No: pass through to CF Pages
    → Yes:
      1. Forward request to CF Pages (don't block response)
      2. Write data point to Analytics Engine (async, non-blocking)
      3. Return response to user
```

Zero perceived latency — the analytics write is asynchronous.

### Analytics Engine Data Model

**Dataset:** `docs_pageviews`

| Field | Type | Content |
|---|---|---|
| `blob1` | string | URL path |
| `blob2` | string | Referrer |
| `blob3` | string | Country code |
| `blob4` | string | UA hash (SHA-256) |
| `blob5` | string | Doc version |
| `blob6` | string | Browser family |
| `blob7` | string | OS family |
| `blob8` | string | Device type |
| `double1` | number | Unix timestamp |

### Cost

- Workers free tier: 100K requests/day
- Analytics Engine free tier: 100K writes/day, 10K queries/day
- At 100K-500K pageviews/month: well within free limits

## Data Warehouse: BigQuery

### GA4 → BigQuery (Native)

- Enable in GA4 Admin → BigQuery Links
- Daily export (free on GA4 standard), streaming export on GA4 360
- Creates event-level dataset in BigQuery with every pageview, event, and user property
- BigQuery free tier: 1 TB query/month, 10 GB storage

### Analytics Engine → BigQuery (Scheduled Export)

- A CF Worker with Cron Trigger runs daily
- Queries Analytics Engine SQL API for the previous day's aggregated data
- Writes to BigQuery via the BigQuery API (service account credentials stored as Worker secret)
- Keeps both consented (GA4) and cookieless (Analytics Engine) data in one warehouse

### Unified Reporting

Both data sources in BigQuery enables:

- **Looker Studio** (free) dashboards combining both datasets
- Custom SQL analysis joining consented and cookieless data
- Long-term retention (Analytics Engine retains 90 days; BigQuery retains indefinitely)
- Future: company identification data (from IP enrichment) can join the same warehouse

## Dashboard

**Analytics Engine data (real-time):** Grafana querying the Analytics Engine SQL API via JSON datasource plugin. Options:

- Grafana Cloud free tier (up to 3 users, 10K metrics)
- Self-hosted Grafana on any VPS

**GA4 data:** GA4 dashboard (native) or Looker Studio connected to BigQuery.

**Unified view:** Looker Studio dashboard querying BigQuery (both GA4 export and Analytics Engine export tables).

## GDPR Compliance

| Component | Cookies | Consent Required | Legal Basis |
|---|---|---|---|
| CookieBot | Sets consent cookie | N/A (consent tool itself) | Legitimate interest |
| GTM + GA4 via sGTM | First-party cookies via metrics.vyos.io | Yes — gated by CookieBot | Consent |
| HubSpot via sGTM | First-party cookies via metrics.vyos.io | Yes — gated by CookieBot | Consent |
| CF Worker + Analytics Engine | None | No | Legitimate interest (no personal data stored) |

Users who reject cookies: GA4 and HubSpot do not fire. CF Worker still captures aggregate pageview data (cookieless, no personal identifiers).

## Future: Company/Org Identification

Deferred to a future implementation. When ready:

- A CF Worker calls an IP-to-company API (Clearbit Reveal, Dealfront, or 6sense) on each request
- Results stored in Analytics Engine and/or pushed to HubSpot CRM via API
- IP-to-company is generally considered legitimate interest (identifies organizations, not individuals)
- Data can flow to BigQuery for unified analysis
- Requires separate API subscription with the enrichment provider

## Implementation Phases

| Phase | What | When |
|---|---|---|
| A | Add GTM + CookieBot snippets to Sphinx templates, configure GTM for docs.vyos.io, add HubSpot tag to sGTM | Immediately (Phase 1 — works on RTD) |
| B | Deploy CF Worker + Analytics Engine | After Cloudflare Migration (Phase 3) |
| C | Enable GA4 → BigQuery export, build Analytics Engine → BigQuery scheduled export | After Phase B |
| D | Build Grafana/Looker Studio dashboards | After Phase C |

## Execution Order (All Six Specs)

| Phase | Specs |
|---|---|
| Phase 1 (parallel) | LLM Adaptation + MyST Migration + Visitor Tracking Phase A (GTM snippets) |
| Phase 2 | Cloudflare Hosting Migration |
| Phase 3 (parallel) | Mobile Performance + Visual Editor + Visitor Tracking Phases B-D |

## Risks

- **GA4 + GDPR:** Even server-side GA4 via sGTM sends data to Google. EU DPAs have scrutinized this. Mitigation: CookieBot consent gating, sGTM proxying through metrics.vyos.io (first-party domain), IP anonymization in GA4 settings.
- **Analytics Engine 90-day retention:** Data beyond 90 days is lost unless exported. Mitigation: daily BigQuery export via scheduled Worker.
- **Analytics Engine has no built-in dashboard.** Mitigation: Grafana or Looker Studio. Adds setup work but is a one-time cost.
- **Bot traffic in cookieless layer.** The CF Worker sees all requests including crawlers. Mitigation: basic UA filtering. Cloudflare Bot Management (paid) available for more accurate filtering.

## Maintenance

- **GTM/CookieBot/sGTM:** Existing infrastructure, maintained by current processes.
- **CF Worker:** Minimal — ~100-200 lines, rarely changes.
- **BigQuery export Worker:** Minimal — runs on Cron Trigger, alert if it fails.
- **Dashboards:** Occasional updates when new metrics or dimensions are needed.
