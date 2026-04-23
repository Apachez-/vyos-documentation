# RST → MyST Markdown Migration + Edit Links

**Date:** 2026-03-29
**Status:** Approved
**Depends on:** None
**Enables:** Visual Editor spec (2026-03-29-visual-editor-design.md)
**Related:** LLM Documentation Adaptation spec (conf.py changes are non-conflicting and can be applied independently)
**Execution:** Parallel with LLM Adaptation (trivial `conf.py` merge conflict in `extensions` list expected). Must complete before Visual Editor work begins.

## Goal

1. Batch-convert all 258 RST documentation files to MyST Markdown, making content accessible to the broader Markdown ecosystem and lowering the contribution barrier.
2. Add "Edit this page" links to every documentation page, enabling drive-by contributions via GitHub's web editor without local Git setup.

## Background

The VyOS documentation uses reStructuredText, which limits contributions to people who know both Git and RST syntax. Markdown is far more widely known. The project already has `myst_parser` enabled in `conf.py`, 7 existing `.md` files, and a `cmdincludemd` directive in `vyos.py` specifically built for the MyST migration path.

No visual editor or CMS platform supports RST. Migrating to MyST Markdown is the prerequisite for any visual editing solution.

## Part 1: RST → MyST Batch Migration

### Tool

[`rst-to-myst`](https://github.com/executablebooks/rst-to-myst) (v0.4.0) with custom directive mappings for VyOS-specific directives.

### Conversion Mapping

| RST Construct | MyST Output | Method |
|---|---|---|
| Standard RST (headings, lists, code blocks, tables, admonitions) | Standard MyST Markdown | Automatic |
| `.. cfgcmd:: <command>` | `` ```{cfgcmd} <command> `` | `parse_content` mapping |
| `.. opcmd:: <command>` | `` ```{opcmd} <command> `` | `parse_content` mapping |
| `:vytask:\`T1234\`` | `` {vytask}`T1234` `` | Automatic (role conversion) |
| `:cfgcmd:\`command\`` | `` {cfgcmd}`command` `` | Automatic (role conversion) |
| `:opcmd:\`command\`` | `` {opcmd}`command` `` | Automatic (role conversion) |
| `.. cmdinclude:: path` | `` ```{cmdincludemd} path `` | Direct mapping + post-process rename |
| `.. toctree::` | `` ```{toctree} `` | Direct |
| `.. code-block:: lang` | Standard fenced code block | Automatic |
| `.. note::` / `.. warning::` etc. | `:::{note}` / `:::{warning}` | `parse_content` mapping |
| `.. figure::` | `` ```{figure} `` | `parse_content` mapping |

### What Stays As-Is

- `_include/*.txt` fragments — these use `{{ var0 }}` template syntax and are currently consumed by the `cmdinclude` directive (RST). After migration, they will be consumed by the `cmdincludemd` directive (MyST). The fragment files themselves are format-agnostic and do not need conversion.
- `_include/vyos-1x/` submodule — XML definitions, unrelated to documentation format.

### `rst-to-myst` Configuration

```yaml
language: en
sphinx: true
extensions:
  - vyos
colon_fences: true
conversions:
  CfgCmdDirective: parse_content
  OpCmdDirective: parse_content
  CfgInclude: direct
  CfgcmdlistDirective: direct
  OpcmdlistDirective: direct
```

Note: The exact directive class names must be verified against `docs/_ext/vyos.py` at conversion time, as `rst-to-myst` uses the docutils directive class names for its mapping configuration.

### Conversion Process

1. Install `rst-to-myst` and configure with VyOS directive mappings.
2. Run `rst2myst convert docs/**/*.rst` in dry-run mode first.
3. Review output for files that fall back to `{eval-rst}` wrapping — these need manual attention.
4. Run batch conversion with `--replace-files` flag.
5. Post-process: rename all `{cmdinclude}` directives to `{cmdincludemd}` in converted files.
6. Build HTML from converted sources: `make html SPHINXOPTS="-W"` (warnings as errors).
7. Diff HTML output before and after migration to catch regressions.
8. Fix any broken cross-references, rendering issues, or directive problems.
9. Remove old `.rst` files once the build is clean and output matches.

### `conf.py` Changes

Enable MyST extensions needed for full feature parity with RST:

```python
myst_enable_extensions = [
    "colon_fence",     # ::: fence syntax for directives
    "deflist",         # definition lists
    "fieldlist",       # field lists (used in metadata)
    "substitution",    # {{ var }} substitutions
]
```

Update `source_suffix` to reflect the new primary format (apply this change only after all `.rst` files have been converted and removed — Sphinx uses the first entry as the default suffix when resolving ambiguous references):

```python
source_suffix = ['.md', '.rst']
```

Verify that the existing `myst-parser==2.0.0` supports all four extensions above. All are supported in myst-parser 2.0.0; no version bump needed.

### Validation

- Build with `-W` flag (warnings as errors) — catches broken references, missing includes.
- Run `sphinx-build -b linkcheck` — validates all external links.
- Run Vale linter — works on both `.rst` and `.md` files.
- Visual diff of HTML output before/after using `diff -r`.

## Part 2: "Edit this Page" Links

### Implementation

Add `html_context` to `conf.py`:

```python
html_context = {
    "display_github": True,
    "github_user": "vyos",
    "github_repo": "vyos-documentation",
    "github_version": "current",
    "conf_py_path": "/docs/",
}
```

The `sphinx_rtd_theme` renders an "Edit on GitHub" link in the top-right of every page automatically when these context variables are set.

### Contributor Workflow

1. Reader spots an error or wants to add something.
2. Clicks "Edit on GitHub" on the page.
3. GitHub opens the `.md` file in its web editor.
4. Contributor edits the Markdown content (familiar syntax post-migration).
5. Writes a commit message, submits as a pull request.
6. Maintainer reviews and merges.

No local Git, no RST knowledge, no special tools needed.

## Risks

- **`rst-to-myst` semi-maintained** (no releases since July 2023). Mitigation: the tool is stable for its purpose; we use it once for batch conversion and don't depend on it ongoing.
- **Custom directive conversion quality.** Mitigation: dry-run first, manual review of files with `{eval-rst}` fallbacks, HTML diff validation.
- **20-40% of files may need manual post-conversion review** based on real-world migration reports. Mitigation: budget time for manual fixes; the build validation catches most issues automatically.

## Maintenance

Post-migration, this is zero-maintenance. All new documentation is written in MyST Markdown. The `rst-to-myst` tool is not a runtime dependency — it's used once and discarded.

**Docker image:** No new runtime dependencies are added. The `myst_enable_extensions` and `html_context` settings are pure `conf.py` configuration. The `vyos/vyos-documentation` Docker image does not need rebuilding for this spec.
