# my-resume

[![Build and Sync Resume](https://github.com/punitpi/my-resume/actions/workflows/build-and-sync.yml/badge.svg)](https://github.com/punitpi/my-resume/actions/workflows/build-and-sync.yml)
[![Download PDF](https://img.shields.io/badge/download-Puneeth--Prakash--Resume.pdf-blue)](https://github.com/punitpi/my-resume/releases/latest/download/Puneeth-Prakash-Resume.pdf)

ATS-friendly LaTeX resume, single source of truth for [typedbyme.puneeth.io](https://typedbyme.puneeth.io).

Two variants live in this repo:
- **`resume.tex`** — the primary resume (two pages, with photo; built on the
  [Awesome-CV](https://github.com/posquit0/Awesome-CV) class, see Credits). Every push on `main`
  is compiled with XeLaTeX, published as `Puneeth-Prakash-Resume.pdf` (workflow artifact and
  rolling GitHub Release, tag `latest`), and copied into the portfolio repository so the live
  site's resume link always serves the current PDF.
- **`resume-plain.tex`** — the original single-column template, kept in the repo as a backup.
  Stable and not actively changing, so it is **not** built automatically on push (see "Local
  development" below to build it, or run the workflow manually with `root_file: resume-plain.tex`).

## Public and private builds

The same `resume.tex` produces two PDFs. The difference is one file, `private.tex`, which holds
the phone number and must never reach a public surface.

Work authorization is **not** private — it is in the public resume. In the DACH market it is a
go/no-go filter recruiters look for, so stating "EU Blue Card (Austria) — no sponsorship
required" up front is the point.

| | Public | Private |
|---|---|---|
| Phone number | absent | present |
| Built by | CI, and locally | CI, and locally |
| Published to | `latest` release, portfolio site | [`punitpi/my-resume-private`](https://github.com/punitpi/my-resume-private) only |
| Source of the extra data | — | `private.tex` locally, `RESUME_PRIVATE_TEX` secret in CI |

`resume.tex` only reads `private.tex` when `\privatebuild` is defined on the command line, so the
public build cannot pick it up by accident. `private.tex` and `resume-private.pdf` are both
git-ignored.

**Locally**, copy `private.example.tex` to `private.tex`, fill in your real details, and build:

```bash
latexmk -xelatex -usepretex='\def\privatebuild{}' -jobname=resume-private resume.tex
```

**In CI**, the private build is reconstructed from the `RESUME_PRIVATE_TEX` secret, compiled, and
pushed to the private mirror repo; `private.tex` is deleted from the runner immediately after the
compile, and the private PDF is never uploaded as an artifact or attached to a release. See
"GitHub Actions / Secrets setup" for the two secrets this needs.

> **Why not just keep the private PDF as a workflow artifact?** On a public repository, workflow
> artifacts and release assets are downloadable by anyone who can see the repo. There is no
> private-artifact setting. The separate private repository is what provides the access control.

## How it works

```
                          push resume.tex
                                 │
                 ┌───────────────┴───────────────┐
                 ▼                               ▼
        public build (no                private build
        private.tex present)          (RESUME_PRIVATE_TEX
                 │                     -> private.tex, then
                 │                      shredded on the runner)
                 ▼                               │
     Puneeth-Prakash-Resume.pdf                  ▼
                 │                     Puneeth-Prakash-Resume.pdf
      ┌──────────┴──────────┐              in punitpi/
      ▼                     ▼             my-resume-private
GitHub Release    static/files/Resume.pdf   (private repo)
   "latest"        in punitpi/typedbyme
(public download)            │
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
latexmk -xelatex resume.tex           # primary resume -> resume.pdf
latexmk -xelatex resume-plain.tex     # backup, classic template
```

This produces `resume.pdf` / `resume-plain.pdf` and leaves auxiliary files (`.aux`, `.log`,
etc.) alongside them — all git-ignored. To clean them up:

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

The workflow (`.github/workflows/build-and-sync.yml`) needs three repository secrets, all set in
**this** repo under **Settings → Secrets and variables → Actions → New repository secret**:

| Secret | Purpose | Failure if missing |
|---|---|---|
| `PORTFOLIO_PAT` | push the public PDF into the portfolio repo | sync step fails; build + release still succeed |
| `PRIVATE_REPO_PAT` | push the private PDF into the private mirror | run fails loudly (by design — a silent skip means a stale private copy) |
| `RESUME_PRIVATE_TEX` | contents of `private.tex` for the CI private build | run fails loudly |

### 1. `PORTFOLIO_PAT`

1. Go to **Settings → Developer settings → Personal access tokens → Fine-grained tokens** and
   generate a new token:
   - **Resource owner:** `punitpi`
   - **Repository access:** Only select repositories → `typedbyme`
   - **Permissions:** Repository → Contents → **Read and write**
   - Set an expiry (fine-grained tokens are capped at 1 year) — note the date, this is the
     pipeline's recurring maintenance item; it will stop syncing once the token expires.
2. Save it as the `PORTFOLIO_PAT` secret.

### 2. `PRIVATE_REPO_PAT`

Same procedure, but scoped to **`my-resume-private`** instead of `typedbyme`. Use a separate
token rather than reusing `PORTFOLIO_PAT`: each token then reaches exactly one repository, so a
leak of either one has a contained blast radius.

### 3. `RESUME_PRIVATE_TEX`

The entire contents of your local `private.tex`, pasted as the secret value. For example:

```latex
\mobile{+43 000 0000000}
```

Update this secret whenever your number changes — it is the CI copy of a file that is
deliberately never committed.

> The `Permissions` → `Contents` option only becomes selectable *after* choosing "Only select
> repositories" and picking the target repo — easy to miss if you skip the repo picker.
4. Push a change to `resume.tex` (or run the workflow manually via **Actions → Build and
   Sync Resume → Run workflow**) to verify the sync end-to-end.

## Outputs

| Artifact | Location |
|---|---|
| Workflow artifact (per run) | Actions run summary → Artifacts → `resume-pdf` |
| Rolling release | [Releases → `latest`](https://github.com/punitpi/my-resume/releases/tag/latest) |
| Stable download link | `https://github.com/punitpi/my-resume/releases/latest/download/Puneeth-Prakash-Resume.pdf` |
| Live portfolio copy | `static/files/Resume.pdf` in [`punitpi/typedbyme`](https://github.com/punitpi/typedbyme) |
| **Private copy** (with phone number) | `Puneeth-Prakash-Resume.pdf` in [`punitpi/my-resume-private`](https://github.com/punitpi/my-resume-private) — private repo, not linked publicly |
| Backup template | `resume-plain.tex` — build locally, not built by CI |

## Credits

- Layout: [Awesome-CV](https://github.com/posquit0/Awesome-CV) by Claud D. Park (LPPL 1.3c),
  vendored as `awesome-cv.cls` with local font-loading patches — see `LICENCE-awesome-cv.txt`.
- Font: [Source Sans 3](https://github.com/adobe-fonts/source-sans) (SIL OFL 1.1), vendored in
  `fonts/` — see `fonts/LICENSE-SourceSans3.txt`.
