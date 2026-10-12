# MuduDB Documentation

This repository contains the Sphinx documentation project for MuduDB.

The documentation is written in Markdown through MyST Parser, organized by database version and language, and built into static HTML for publication.

## Layout

```text
mududb.document/
  source/
    conf.py
    _static/
      custom.css
    versions/
      v0.1/
        en/
          index.md
          ...
        zh_cn/
          index.md
          ...
  scripts/
    build_docs.py
  .github/
    workflows/
      docs.yml
```

Version and language are both part of the source path:

```text
source/versions/<db-version>/<language>/
```

Current starter content:

- `v0.1/en`
- `v0.1/zh_cn`

Add future database versions by copying an existing version directory, for example:

```powershell
Copy-Item -Recurse source\versions\v0.1 source\versions\v0.2
```

Then update `source/versions/versions.json`.

## Requirements

Use Python 3.10 or newer.

### Virtual Environment Setup

Create and activate a virtual environment before installing dependencies.

**Linux / macOS:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

**Windows (PowerShell):**

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

**Windows (CMD):**

```cmd
python -m venv .venv
.venv\Scripts\activate.bat
```

### Install Dependencies

With the virtual environment activated:

```bash
python -m pip install -r requirements.txt
```

## Build

Build all configured versions and languages:

```bash
python scripts/build_docs.py
```

Build one version and language:

```bash
python scripts/build_docs.py --version v0.1 --language en
python scripts/build_docs.py --version v0.1 --language zh_cn
```

Generated HTML is written to:

```text
build/html/<version>/<language>/
```

The script also creates:

- `build/html/index.html`, redirecting to the default version and language.
- `build/html/.nojekyll`, for GitHub Pages compatibility.

## Local Preview

After building:

```bash
python -m http.server 8000 --directory build/html
```

Open:

```text
http://127.0.0.1:8000/
```

## Writing Guide

Write pages in Markdown (`.md`). MyST extensions are enabled for common documentation features:

- fenced code blocks
- tables
- definition lists
- task lists
- admonitions
- colon fences
- field lists
- smart quotes and replacements

Use Sphinx toctrees in Markdown:

````md
```{toctree}
:maxdepth: 2

getting-started/index
reference/index
```
````

Each page should belong to one version and one language. Do not share mutable text files across languages; keep translation ownership explicit.

## Version Maintenance

Use this policy for database releases:

- Patch-level documentation fixes can be made in the affected version directory.
- New feature documentation should be added to the next active version directory.
- When a DB release becomes historical, keep its directory intact and avoid restructuring it.
- If a page is removed in a future version, leave the older version unchanged.

Update `source/versions/versions.json` when adding or retiring maintained versions:

```json
{
  "default_version": "v0.1",
  "default_language": "zh_cn",
  "versions": ["v0.1"],
  "languages": {
    "en": "English",
    "zh_cn": "简体中文"
  }
}
```

## Publication Workflow

The workflow at `.github/workflows/docs.yml` builds the final static HTML and publishes only generated artifacts.

The workflow publishes generated HTML to the `docs` directory in `https://github.com/mududb/mududb.github.io`.

Configure these repository variables/secrets in the documentation repository:

- Secret `PAGES_DEPLOY_TOKEN`: GitHub token with access to `mududb/mududb.github.io`.
  Fine-grained tokens must include the target repository and `Contents: Read and write`. Classic tokens need `repo` access.

The workflow publishes `build/html` into `docs/` in `mududb/mududb.github.io`. Source files from this repository are not published.
Publishing is manual: run the workflow with `command=publish`. Pushes to `main` run the build job only.

## Manual Release Checklist

1. Update or copy the version directory under `source/versions`.
2. Update `source/versions/versions.json`.
3. Build locally with `python scripts/build_docs.py`.
4. Inspect `build/html/index.html` and one language page.
5. Push to `main` to run the build check.
6. Run the workflow manually with `command=publish` to publish generated HTML.

## Documentation baseline

The v0.1 English and Chinese pages were reconciled on 2026-10-12 against
MuduDB source commit `614413db`, with `scuptio/mududb.document` commit
`c9451e7` as a reference. Source implementation takes precedence when the
reference documentation differs. This records a source-checkout baseline,
not the availability of a published binary release.

When updating these pages, check configuration loading and defaults in
`mudu_runtime`, CLI arguments in `mudu_cli` and the packaging tools, syscall
shapes in `mudu_binding/wit/uni-syscall.wit`, runtime isolation in
`runtime_wasm`, SQL behavior in the parser/binder and integration tests,
and SDK workflows in `doc/dev/sdk_distribution.md`. Update both languages
and their toctrees, then run `python scripts/build_docs.py --clean`.
