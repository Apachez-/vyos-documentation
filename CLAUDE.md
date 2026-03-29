# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

VyOS network OS documentation, built with Sphinx from reStructuredText sources. Hosted at https://docs.vyos.io. The `current` branch tracks VyOS 1.5.x (circinus) rolling release.

## Build Commands

All build commands run from the `docs/` directory.

### Local (requires Sphinx installed via `pip install -r requirements.txt`)

```bash
cd docs
make html              # Build HTML to docs/_build/html/
make livehtml          # Live preview at http://localhost:8000
make pdf               # Generate PDF
```

### Docker (preferred)

```bash
# Build HTML
docker run --rm -it -v "$(pwd)":/vyos -w /vyos/docs -e GOSU_UID=$(id -u) -e GOSU_GID=$(id -g) vyos/vyos-documentation make html

# Live preview
docker run --rm -it -p 8000:8000 -v "$(pwd)":/vyos -w /vyos/docs -e GOSU_UID=$(id -u) -e GOSU_GID=$(id -g) vyos/vyos-documentation make livehtml
```

### Linting (Vale)

```bash
# Lint all files
docker run --rm -it -v "$(pwd)":/vyos -w /vyos/docs -e GOSU_UID=$(id -u) -e GOSU_GID=$(id -g) vyos/vyos-documentation vale .

# Lint a single file
docker run --rm -it -v "$(pwd)":/vyos -w /vyos/docs -e GOSU_UID=$(id -u) -e GOSU_GID=$(id -g) vyos/vyos-documentation vale quick-start.rst
```

## Architecture

### Documentation Source

- All RST source files live under `docs/` following the VyOS CLI hierarchy (e.g., `set firewall zone` → `docs/configuration/firewall/zone.rst`)
- Main index: `docs/index.rst`
- Sphinx config: `docs/conf.py`

### Custom Sphinx Extensions (`docs/_ext/`)

- **vyos.py** — Core extension defining custom directives and roles:
  - `.. cfgcmd::` — Document configuration commands
  - `.. opcmd::` — Document operational commands
  - `.. cfgcmdlist::` / `.. opcmdlist::` — Generate command lists with coverage status
  - `.. cmdinclude::` — Include RST with variable substitution (var0-var9)
  - Roles: `:vytask:` (links to task tracker), `:cfgcmd:`, `:opcmd:` (inline command references)
- **testcoverage.py** — Compares documented commands against XML definitions and actual VyOS commands
- **releasenotes.py** — Release notes integration

### Shared Fragments (`docs/_include/`)

- `interface-*.txt` files contain shared RST blocks (e.g., common interface options) included via `.. cmdinclude::` with `var0`–`var9` substitution
- `vyos-1x/` submodule contains VyOS source XML definitions used for command coverage analysis

### Additional Extensions

- **autosectionlabel.py** — Custom wrapper enabling cross-document section references via `autosectionlabel_prefix_document = True`
- Both `.rst` and `.md` (MyST Markdown) are valid source formats

## RST Writing Conventions

- **Language**: American English
- **Line length**: Max 80 characters (URLs exempt)
- **Indentation**: 2 spaces
- **Heading hierarchy** (top to bottom): `#####`, `*****`, `=====`, `-----`, `^^^^^`, `""""""`
- Every RST file must start with the `#####` level title
- Leave a newline before and after headers
- Quote commands/filenames with double backticks; use literal blocks for longer snippets
- Add `:lastproofread: YYYY-MM-DD` metadata to track proofreading currency (warns if >365 days old)

### Page Structure

Each documentation page should follow this order:
1. Theoretical information (what/why/when)
2. Configuration description (using `cfgcmd` directives)
3. Configuration examples with topology
4. Known issues and workarounds
5. Debugging procedures

### Update `index.rst` when adding new pages.
