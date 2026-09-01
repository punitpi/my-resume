# my-resume

LaTeX resume with CI-driven build, GitHub Release, and auto-sync to the portfolio repo
(`punitpi/typedbyme`, `static/files/Resume.pdf`, capital R — case matters, the live site links
that exact path).

## Commands

```bash
latexmk -xelatex resume-awesome.tex   # primary resume (needs a local TeX install - see README)
latexmk -xelatex resume.tex           # backup template
latexmk -c                            # clean aux files
```

Full local setup (MacTeX/TeX Live, VS Code LaTeX Workshop) and GitHub Secrets configuration are
in `README.md` — don't duplicate those steps here.

## Skills

- `verify-resume-build` — after any push to `.tex`/`.cls`, watches the CI run, downloads the
  PDF, and checks page count / ATS text extraction / render. Use instead of re-deriving the
  `gh run watch` → `gh run download` → `pdfinfo`/`pdftotext` sequence by hand.
- `add-resume-entry` — adds a job/project/education entry with the correct `cventry` argument
  order for that section (see the gotcha below). User-invoked only.

## Structure

- `resume-awesome.tex` / `resume.tex` — the two resume sources (see below).
- `awesome-cv.cls` — vendored + locally patched Awesome-CV class (LPPL); `LICENCE-awesome-cv.txt`
  is its license.
- `fonts/` — vendored Source Sans 3 OTF weights (SIL OFL, `fonts/LICENSE-SourceSans3.txt`),
  referenced by explicit path for `resume-awesome.tex`.
- `images/` — profile photo used in the header.
- `.github/workflows/build-and-sync.yml` — the only CI workflow; needs a `PORTFOLIO_PAT` repo
  secret to reach the sync step (see README's "GitHub Actions / Secrets setup"). Without it, the
  build/artifact/release steps still succeed — only the portfolio-sync step fails.

## Primary vs backup

- **`resume-awesome.tex`** is the shipped resume. Every push to it (or `awesome-cv.cls`,
  `fonts/**`, `images/**`) triggers `.github/workflows/build-and-sync.yml`: compile → artifact →
  rolling GitHub Release (tag `latest`) → sync into the portfolio repo → portfolio's own Pages
  workflow redeploys the live site.
- **`resume.tex`** is a backup, single-column template. Not built by CI (excluded from the
  workflow's path filter on purpose, to save CI time on a stable file). Build it locally with
  `latexmk -xelatex resume.tex`, or run the workflow manually with `root_file: resume.tex`.

## No local LaTeX toolchain

This machine has no `xelatex`/`pdflatex`/`latexmk` and no Docker/Podman. Every `.tex` or `.cls`
change must be verified via CI: push, `gh run watch <id> --repo punitpi/my-resume --exit-status`,
then `gh run download <id> --name resume-awesome-pdf --dir /tmp/...` and inspect with
`pdfinfo`/`pdftotext`/`pdftoppm` (poppler, installed via `brew install poppler`). Each round-trip
is ~2 minutes. Batch independent hypotheses into one push rather than testing one variable at a
time when possible.

## `awesome-cv.cls` gotchas (vendored, locally patched)

- **`\cventry{#1}{#2}{#3}{#4}{#5}`**: `#2` renders bold on line 1, `#1` renders small-caps on
  line 2 — the *opposite* of reading order. For Experience: `#1`=role, `#2`=company (company
  prominent). For Projects: `#1`=tech stack, `#2`=project name (name prominent) — easy to get
  backwards, verify against a render, not just the source.
- **FontAwesome6Brands icons have no ToUnicode/cmap** (`pdffonts` shows `uni=no` for that subfont
  only). `accsupp`'s `ActualText` mechanism (used by `\faAlt`) does not reliably attach to those
  glyphs via XeLaTeX's PDF backend — GitHub/LinkedIn icons rendered fine visually but were
  silently absent from `pdftotext` output. Fixed via a local `\faAltVisible` macro (always-visible
  plain text instead of relying on ActualText) — used only for GitHub/LinkedIn. `Renderer=HarfBuzz`
  disabling was tried first and was NOT the cause; left disabled anyway rather than re-testing an
  unrelated change against an already-verified document.
- **Fonts are vendored by explicit file path**, not family name (`Path=./fonts/`,
  `Extension=.otf`, `UprightFont=*-Regular` etc.) — avoids a fontconfig family-name lookup that's
  unreliable in the CI container. Source Sans 3 static OTF weights (not TTF — TTF worked fine too,
  this was a red herring during debugging) from `adobe-fonts/source-sans` releases, SIL OFL 1.1.
  Roboto (upstream's header font) has no static-weight release to vendor; mapped to the same
  Source Sans 3 files instead.
- **`\acvSectionContentTopSkip`** is paired with a fixed `-3mm`/`-2mm` `\vspace` at each call site
  in the class. Setting it much below ~2mm nets a large enough negative space to pull paragraph
  text up into the section-title line above it (hit this with `cvparagraph`/Summary). Keep it at
  2mm or above when tightening for page-fit.
- **`cvhonors`** (3-column: date | title | location) looks visually disconnected when used for a
  single entry next to a `cventries` table (2-column) in the same section — different column
  proportions. Prefer a second `cventry` in the same `cventries` block for consistency, not
  `cvhonors`, unless there are several honor/award entries.
- ATS verification that actually matters: `pdftotext -raw file.pdf - | grep -oE 'punitpi|ppuneeth'`
  (or other identifying strings) — a clean visual render does not prove the text layer is intact.

## CI trigger scope

`awesome-cv.cls` is in the path filter (`**/*.cls`), so *any* edit to it — including a
comment-only change — triggers a full rebuild + release + portfolio sync. `resume.tex`, README,
and other non-listed files do not. This is intentional (a missed real change is worse than an
occasional harmless rebuild), but worth knowing before assuming a `.cls` comment tweak is free.

## GitHub Actions

- `xu-cheng/latex-action@v4`, `latexmk_use_xelatex: true`. Alpine-based image by default, no
  system fonts — `\setmainfont` on a named family (even TeX Gyre) risks a fontconfig lookup
  failure; either omit it (Latin Modern fallback) or vendor by file path.
- `softprops/action-gh-release@v3` on a fixed `tag_name` updates the existing release
  (`overwrite_files` defaults true) rather than erroring or duplicating — confirmed by pushing
  twice and checking the asset was replaced, not just the tag.
- Portfolio sync pushes with `PORTFOLIO_PAT` (fine-grained, `Contents: read/write`, scoped to
  `typedbyme` only) instead of the default `GITHUB_TOKEN` — PAT-authenticated pushes trigger
  downstream workflows; `GITHUB_TOKEN` pushes are suppressed to prevent recursive triggers. This
  is what makes the portfolio's Pages workflow fire automatically.
- Fine-grained PAT creation: the Permissions/Contents option only becomes selectable *after*
  choosing "Only select repositories" and picking the target repo — it's easy to miss if the repo
  picker step is skipped.
