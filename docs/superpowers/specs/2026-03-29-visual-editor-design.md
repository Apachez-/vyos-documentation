# Visual Documentation Editor

**Date:** 2026-03-29
**Status:** Approved
**Depends on:** RST → MyST Migration (2026-03-29-rst-to-myst-migration-design.md)
**Related:** LLM Documentation Adaptation spec (independent, no shared infrastructure)
**Execution:** Starts after MyST Migration is merged. Can run in parallel with LLM Adaptation if that hasn't completed yet.

## Goal

Deploy a web-based visual editor for VyOS documentation that enables non-technical contributors to edit content without Git knowledge. Edits go through a staging buffer where maintainers review and approve before changes reach the Git repository.

## Target Users

- **Network engineers** who know VyOS but aren't developers — can fix errors, add examples, update outdated info
- **Drive-by contributors** — quick fixes via "Edit this page" links (handled by the migration spec, not this editor)
- **Dedicated docs contributors** — write substantial new content with live preview

## Architecture

```
┌─────────────────────────────────┐
│  Antmicro MyST Editor (Preact)  │  ← Client-side, served as static files
│  - Split-pane editor + preview  │
│  - MyST directive rendering     │
│  - File browser / page picker   │
└──────────────┬──────────────────┘
               │ REST API
┌──────────────▼──────────────────┐
│  Staging API (lightweight)      │  ← Node.js or Python service
│  - Load page content from Git   │
│  - Save drafts to staging DB    │
│  - List pending edits for review│
│  - Auth via Auth0 (JWT)         │
│  - Roles: contributor, maintainer│
└──────────────┬──────────────────┘
               │
       ┌───────┴───────┐
       │               │
┌──────▼──────┐ ┌──────▼──────┐
│ Staging DB  │ │  Git repo   │
│ (SQLite)    │ │ (source of  │
│ - drafts    │ │  truth)     │
│ - metadata  │ │             │
└─────────────┘ └─────────────┘
```

## Components

### 1. Editor Frontend

**Technology:** [Antmicro MyST Editor](https://github.com/antmicro/myst-editor) (Preact, Apache 2.0)

The only web-based editor with native MyST directive support. Renders `{cfgcmd}`, `{opcmd}`, and other custom directives meaningfully in the preview pane.

**Features to use:**
- Split-pane editor with live MyST preview
- Colon fence and directive syntax highlighting
- Real-time collaborative editing (YJS) — optional, enable later if needed

**Features to build:**
- File browser / page picker — list documentation pages, search by title
- "Edit" button that loads a page's current content from the staging API
- "Save draft" and "Submit for review" actions
- Link back to the rendered docs page

**Deployment:** Static files served alongside the docs or on a subdomain (e.g., `edit.docs.vyos.io`).

### 2. Staging API

**Technology:** Lightweight HTTP service (Node.js or Python/FastAPI — choose at implementation time).

**Endpoints:**

```
GET    /api/pages                  — List all documentation pages (file paths + titles)
GET    /api/pages/:path            — Get current content of a page from Git
POST   /api/drafts                 — Create a new draft
GET    /api/drafts                 — List drafts (filtered by status, author)
GET    /api/drafts/:id             — Get a specific draft with diff
PATCH  /api/drafts/:id             — Update draft content or status
POST   /api/drafts/:id/submit      — Submit draft for review
POST   /api/drafts/:id/review      — Approve, reject, or request changes (maintainer)
POST   /api/drafts/:id/commit      — Commit approved draft to Git and create PR (maintainer)
```

**Git operations:** The API clones/pulls the documentation repo server-side. When a maintainer approves and commits:
1. Creates a branch from `current`
2. Writes the updated file content
3. Commits with the contributor's name in the commit message
4. Creates a PR via the GitHub API using a GitHub App token
5. Optionally auto-merges if the maintainer chose "approve and merge"

### 3. Authentication (Auth0)

Uses the existing Auth0 tenant. Two roles:

- **Contributor** — can browse pages, create/edit drafts, submit for review
- **Maintainer** — all contributor permissions plus: review drafts, approve/reject, trigger Git commits

Auth flow:
1. User clicks "Sign in" in the editor
2. Redirected to Auth0 login (supports whatever identity providers are configured)
3. Returns with JWT containing user ID, name, and role
4. API validates JWT on every request

### 4. Staging Database

SQLite, single file, minimal schema:

```sql
CREATE TABLE drafts (
    id            TEXT PRIMARY KEY,     -- UUID
    file_path     TEXT NOT NULL,        -- e.g., "configuration/firewall/zone.md"
    original_sha  TEXT NOT NULL,        -- Git SHA when editing started
    content       TEXT NOT NULL,        -- full file content after edits
    author_id     TEXT NOT NULL,        -- Auth0 user ID
    author_name   TEXT,                 -- display name
    note          TEXT,                 -- contributor's description of the change
    status        TEXT NOT NULL,        -- draft | submitted | approved | rejected | committed
    created_at    TIMESTAMP NOT NULL,
    updated_at    TIMESTAMP NOT NULL
);

CREATE TABLE reviews (
    id            TEXT PRIMARY KEY,     -- UUID
    draft_id      TEXT NOT NULL REFERENCES drafts(id),
    reviewer_id   TEXT NOT NULL,        -- Auth0 user ID (maintainer)
    action        TEXT NOT NULL,        -- approve | reject | request_changes
    comment       TEXT,
    created_at    TIMESTAMP NOT NULL
);
```

**Conflict detection:** When a maintainer approves a draft, the API compares `original_sha` against the current Git HEAD for that file. If the file changed since the contributor started editing, the maintainer sees a warning with a diff and can:
- Merge manually
- Ask the contributor to re-edit against the updated version
- Override and commit anyway

**Draft lifecycle:** `draft` → `submitted` → (`approved` → `committed`) or (`rejected`) or (`request_changes` → `draft`)

## Deployment

The system has two deployable components:

1. **Editor frontend** — static files, deploy anywhere (Cloudflare Pages, Nginx, S3)
2. **Staging API** — lightweight service requiring persistent storage (SQLite file) and network access to GitHub API

**Default deployment:** A single VPS running the API with SQLite, serving the static frontend via Nginx on a subdomain (e.g., `edit.docs.vyos.io`). Can scale to separate services later.

## Risks

- **Antmicro MyST Editor maturity.** It's an internal tool made public, not a widely-adopted product. Mitigation: evaluate during implementation; if it doesn't meet needs, fall back to a Monaco/CodeMirror editor with a simpler preview.
- **Custom directive preview.** The editor may not render `{cfgcmd}` blocks out of the box — may need to register custom renderers. Mitigation: acceptable to show directive blocks as styled code fences initially; full rendering is a nice-to-have.
- **Staging API is custom code.** Mitigation: keep it minimal — the API is a thin CRUD layer over SQLite plus Git operations. Estimated ~500-800 lines of code.

## Maintenance

- **Editor frontend:** Track Antmicro MyST Editor releases, update periodically.
- **Staging API:** Minimal — handles CRUD and Git operations. Auth0 and GitHub App tokens need rotation per their policies.
- **Database:** SQLite, no administration. Old committed/rejected drafts can be pruned periodically.
