# my-resume

[![Build and Sync Resume](https://github.com/punitpi/my-resume/actions/workflows/build-and-sync.yml/badge.svg)](https://github.com/punitpi/my-resume/actions/workflows/build-and-sync.yml)
[![Download PDF](https://img.shields.io/badge/download-resume.pdf-blue)](https://github.com/punitpi/my-resume/releases/latest/download/resume.pdf)

ATS-friendly LaTeX resume, single source of truth for [typedbyme.puneeth.io](https://typedbyme.puneeth.io).
Every push to `resume.tex` on `main` is compiled with XeLaTeX, published as a workflow artifact
and as a rolling GitHub Release (tag `latest`), and copied into the portfolio repository so the
live site's resume link always serves the current PDF.

## How it works

```
push resume.tex ──► CI compiles with XeLaTeX ──► resume.pdf
                                                      │
                          ┌───────────────────────────┼───────────────────────────┐
                          ▼                            ▼
              GitHub Release "latest"        static/files/Resume.pdf
              (stable download URL)          in punitpi/typedbyme
                                                        │
                                                        ▼
                                          typedbyme's own Pages workflow
                                          rebuilds and redeploys the site
```

The portfolio sync pushes with a Personal Access Token rather than the default `GITHUB_TOKEN`,
because PAT-authenticated pushes trigger downstream workflows (the default token's pushes are
suppressed to prevent infinite loops). That's what makes the portfolio's own deploy fire
automatically.

## Local development

### 1. Install a TeX distribution with XeLaTeX

- **macOS:** [MacTeX](https://www.tug.org/mactex/) (`brew install --cask mactex`), or the smaller
  [BasicTeX](https://www.tug.org/mactex/morepackages.html) plus `sudo tlmgr install fontspec titlesec enumitem`.
- **Windows/Linux:** [TeX Live](https://www.tug.org/texlive/) (`sudo apt install texlive-full` on
  Debian/Ubuntu, or the TeX Live installer on Windows).

Verify the install:

```bash
xelatex --version
```

### 2. Build the PDF

```bash
latexmk -xelatex resume.tex
```

This produces `resume.pdf` and leaves auxiliary files (`.aux`, `.log`, etc.) alongside it — all
git-ignored. To clean them up:

```bash
latexmk -c
```

### 3. Editing in VS Code (optional)

Install the [LaTeX Workshop](https://marketplace.visualstudio.com/items?itemName=James-Yu.latex-workshop)
extension, then add to your workspace `.vscode/settings.json`:

```jsonc
{
  "latex-workshop.latex.recipes": [
    {
      "name": "xelatex",
      "tools": ["xelatex"]
    }
  ],
  "latex-workshop.latex.tools": [
    {
      "name": "xelatex",
      "command": "xelatex",
      "args": [
        "-synctex=1",
        "-interaction=nonstopmode",
        "-file-line-error",
        "%DOC%"
      ]
    }
  ]
}
```

Build with `Ctrl+Alt+B` (or `Cmd+Alt+B` on macOS); preview with the split-view PDF viewer the
extension provides.

## GitHub Actions / Secrets setup

The workflow (`.github/workflows/build-and-sync.yml`) needs one repository secret to sync the
built PDF into the portfolio repo:

1. In [`punitpi/typedbyme`](https://github.com/punitpi/typedbyme), go to **Settings → Developer
   settings → Personal access tokens → Fine-grained tokens** and generate a new token:
   - **Resource owner:** `punitpi`
   - **Repository access:** Only select repositories → `typedbyme`
   - **Permissions:** Repository → Contents → **Read and write**
   - Set an expiry (fine-grained tokens are capped at 1 year) — note the date, this is the
     pipeline's one recurring maintenance item; it will silently stop syncing once the token
     expires.
2. Copy the generated token.
3. In **this** repo (`punitpi/my-resume`), go to **Settings → Secrets and variables → Actions →
   New repository secret**:
   - **Name:** `PORTFOLIO_PAT`
   - **Value:** the token from step 2
4. Push a change to `resume.tex` (or run the workflow manually via **Actions → Build and Sync
   Resume → Run workflow**) to verify the sync end-to-end.

## Outputs

| Artifact | Location |
|---|---|
| Workflow artifact (per run) | Actions run summary → Artifacts → `resume-pdf` |
| Rolling release | [Releases → `latest`](https://github.com/punitpi/my-resume/releases/tag/latest) |
| Stable download link | `https://github.com/punitpi/my-resume/releases/latest/download/resume.pdf` |
| Live portfolio copy | `static/files/Resume.pdf` in [`punitpi/typedbyme`](https://github.com/punitpi/typedbyme) |
