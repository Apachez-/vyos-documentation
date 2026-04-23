# LLM Documentation Adaptation Design

**Date:** 2026-03-29
**Status:** Approved
**Approach:** Hybrid — automated Sphinx extension + curated `llms.txt`
**Depends on:** None (independent of other specs; `conf.py` changes are non-conflicting)
**Related:** RST → MyST Migration spec, Cloudflare Hosting spec
**Execution:** Parallel with MyST Migration (trivial `conf.py` merge conflict in `extensions` list expected)

## Goal

Make VyOS documentation consumable by LLMs for two use cases:

1. **Structured reference data** — enable LLMs (ChatGPT, Claude, etc.) to answer VyOS configuration questions accurately
2. **AI search discoverability** — make docs indexable by AI search products (Perplexity, Bing Chat, Google AI Overviews)

Target audience: both end users (configuration/operations) and developers/contributors equally.

## Design

### 1. Sphinx Extension: `sphinx-llms-txt`

Add [`sphinx-llms-txt`](https://pypi.org/project/sphinx-llms-txt/) (v0.7.1+) to the Sphinx build pipeline.

**What it generates:**

- `llms-full.txt` — entire documentation concatenated into a single markdown file for bulk LLM ingestion

**Why this extension over `sphinx-llm` (NVIDIA):**

- More mature (v0.7.1 vs v0.3.0)
- Lighter weight — post-processes during normal build rather than running a parallel markdown build
- Supports size limit configuration (useful for 258-file doc set)
- Available on conda-forge

### 2. Curated `llms.txt`

Hand-maintained file at `docs/_html_extra/llms.txt`, served at site root via the existing `html_extra_path` mechanism.

**Content:**

```markdown
# VyOS

> VyOS is a free, open-source network operating system based on Debian GNU/Linux.
> It provides routing, firewall, and VPN functionality via a unified CLI using
> `set`/`delete`/`show` command hierarchy. Current rolling release is 1.5.x (circinus).

VyOS configuration follows a tree structure. All configuration commands start with
`set` (to add/change) or `delete` (to remove). Changes are staged and applied with
`commit`. The CLI hierarchy maps directly to the documentation structure.

## Quick Start

- [Quick Start Guide](https://docs.vyos.io/en/latest/quick-start.html): Minimal setup walkthrough
- [CLI Overview](https://docs.vyos.io/en/latest/cli.html): Command-line interface usage

## Configuration

- [Firewall](https://docs.vyos.io/en/latest/configuration/firewall/index.html): Zone-based firewall, rules, groups
- [Interfaces](https://docs.vyos.io/en/latest/configuration/interfaces/index.html): Ethernet, bonding, bridge, VLAN, tunnel, wireless
- [Protocols](https://docs.vyos.io/en/latest/configuration/protocols/index.html): BGP, OSPF, IS-IS, static routing, MPLS
- [VPN](https://docs.vyos.io/en/latest/configuration/vpn/index.html): IPsec, OpenVPN, WireGuard, L2TP, PPTP
- [NAT](https://docs.vyos.io/en/latest/configuration/nat/index.html): Source NAT, destination NAT, NAT66
- [System](https://docs.vyos.io/en/latest/configuration/system/index.html): DNS, NTP, syslog, users, task scheduler
- [High Availability](https://docs.vyos.io/en/latest/configuration/highavailability/index.html): VRRP
- [Load Balancing](https://docs.vyos.io/en/latest/configuration/loadbalancing/index.html): WAN and reverse proxy
- [Containers](https://docs.vyos.io/en/latest/configuration/container/index.html): Podman-based container support
- [PKI](https://docs.vyos.io/en/latest/configuration/pki/index.html): Certificate management
- [Policy](https://docs.vyos.io/en/latest/configuration/policy/index.html): Route maps, prefix lists, access lists
- [Traffic Policy](https://docs.vyos.io/en/latest/configuration/trafficpolicy/index.html): QoS and shaping
- [Service](https://docs.vyos.io/en/latest/configuration/service/index.html): DHCP, DNS forwarding, SNMP, SSH, HTTPS API
- [VRF](https://docs.vyos.io/en/latest/configuration/vrf/index.html): Virtual routing and forwarding

## Operations

- [Operational Commands](https://docs.vyos.io/en/latest/operation/index.html): Show, monitor, restart commands

## Installation

- [Installation Guide](https://docs.vyos.io/en/latest/installation/index.html): Bare metal, virtual, cloud deployments

## Automation

- [Automation](https://docs.vyos.io/en/latest/automation/index.html): Ansible, Terraform, HTTP API, NETCONF

## Configuration Examples

- [Blueprints](https://docs.vyos.io/en/latest/configexamples/index.html): Real-world topology examples

## Optional

- [Contributing](https://docs.vyos.io/en/latest/contributing/index.html): Development workflow
- [VPP](https://docs.vyos.io/en/latest/vpp/index.html): Vector Packet Processing integration
```

**Design decisions:**

- Preamble explains `set`/`delete`/`commit` model — the most critical context for LLMs answering VyOS questions
- Sections mirror the documentation structure
- `## Optional` separates contributor/VPP content that most end-user queries won't need
- URLs use `/en/latest/` (ReadTheDocs canonical path for current branch). When the Cloudflare Pages migration happens, update URLs to `/en/1.5/` per that spec's URL structure.

### 3. robots.txt Update

Replace `docs/_html_extra/robots.txt` with explicit AI bot allowances:

```
User-agent: *
Allow: /

# AI Training Crawlers
User-agent: GPTBot
Allow: /

User-agent: ClaudeBot
Allow: /

User-agent: Google-Extended
Allow: /

User-agent: CCBot
Allow: /

User-agent: PerplexityBot
Allow: /

# AI Search/Retrieval
User-agent: ChatGPT-User
Allow: /

User-agent: Claude-SearchBot
Allow: /

User-agent: Claude-User
Allow: /

User-agent: OAI-SearchBot
Allow: /

User-agent: Perplexity-User
Allow: /

Sitemap: https://docs.vyos.io/sitemap.xml
```

**Design decisions:**

- Explicitly allow all AI bots (training and retrieval) — VyOS is open-source; broad LLM awareness benefits the project
- Named entries make intent explicit and allow selective blocking later if needed
- Removed `atlassian-bot` special case — covered by wildcard

### 4. Sitemap Generation

Add `sphinx-sitemap` to generate the `sitemap.xml` already referenced by `robots.txt`. This spec owns the `sphinx-sitemap` introduction; the Cloudflare hosting spec references it during DNS cutover but does not add it.

- Requires `html_baseurl = 'https://docs.vyos.io/en/latest/'` in `conf.py` (update to `/en/1.5/` when Cloudflare migration happens)
- ReadTheDocs serves the generated sitemap automatically

### 5. Build Pipeline Changes

**`requirements.txt`** — add (pin to current stable versions at implementation time):

```
sphinx-llms-txt>=0.7.1
sphinx-sitemap>=2.6.0
```

**`conf.py`** — add to extensions:

```python
'sphinx_llms_txt',
'sphinx_sitemap',
```

Add configuration:

```python
html_baseurl = 'https://docs.vyos.io/en/latest/'
```

**No changes to:**

- Custom extensions (`vyos.py`, `testcoverage.py`, `autosectionlabel.py`, `releasenotes.py`)
- RST source files
- Makefile (extensions hook into standard `make html`)

**Docker image:** The `vyos/vyos-documentation` Docker image installs dependencies from `requirements.txt`. Adding `sphinx-llms-txt` and `sphinx-sitemap` to `requirements.txt` is sufficient — no Docker image rebuild needed unless dependencies are baked in at image build time. Verify at implementation time.

**`conf.py` coordination:** The changes in this spec (adding extensions, `html_baseurl`) are to different config keys than the MyST migration spec's changes (`myst_enable_extensions`, `source_suffix`, `html_context`). They are non-conflicting and can be applied in any order.

## Maintenance

- **`llms.txt`**: Update when major documentation sections are added or reorganized. Expected frequency: a few times per year.
- **`llms-full.txt`**: Fully automated, regenerated on every build.
- **`robots.txt`**: Update when new AI crawlers emerge or policy changes.
- **`sitemap.xml`**: Fully automated.

## Hosting

Designed for the current ReadTheDocs setup. ReadTheDocs has built-in support for serving `llms.txt` and `llms-full.txt`. When the Cloudflare Pages migration happens (per the existing hosting spec), the static files will work as-is — they're just files in the build output. However, `html_baseurl` and the curated `llms.txt` URLs must be updated from `/en/latest/` to `/en/1.5/` to match the Cloudflare URL structure and avoid redirect chains.

Note: After the RST → MyST migration, `llms-full.txt` content may change slightly since `sphinx-llms-txt` post-processes built HTML output. Verify output quality after migration.

## Industry Reference

This approach matches what Stripe, Vercel, Cloudflare, and Next.js use: a curated `llms.txt` entry point with automated full-content generation. Stripe notably includes LLM-specific instructions in their `llms.txt` (e.g., "check npm registry for latest versions rather than relying on memorized version numbers"), which informed our VyOS preamble explaining the `set`/`delete`/`commit` model.
