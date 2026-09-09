# my-resume

LaTeX resume with CI-driven build, GitHub Release, and auto-sync to the portfolio repo
(`punitpi/typedbyme`, `static/files/Resume.pdf`, capital R — case matters, the live site links
that exact path).

## Commands

```bash
export PATH=/usr/local/texlive/2026/bin/universal-darwin:$PATH   # TeX Live is installed but NOT on PATH
latexmk -xelatex resume.tex                                       # primary resume -> resume.pdf
latexmk -xelatex -usepretex='\def\privatebuild{}' -jobname=resume-private resume.tex   # + phone/permit
latexmk -xelatex resume-plain.tex                                 # backup template
latexmk -c                                                        # clean aux files
```

Full local setup (VS Code LaTeX Workshop) and GitHub Secrets configuration are in `README.md` —
don't duplicate those steps here.

## Skills

- `verify-resume-build` — after any push to `resume.tex`/`.cls`, watches the CI run, downloads the
  PDF, and checks page count / ATS text extraction / render. Use instead of re-deriving the
  `gh run watch` → `gh run download` → `pdfinfo`/`pdftotext` sequence by hand.
- `add-resume-entry` — adds a job/project/education/award entry with the correct `cventry` argument
  order for that section (see the gotcha below). User-invoked only.

## Structure

- `resume.tex` — the shipped resume (Awesome-CV layout, two pages, photo). CI publishes it as
  `Puneeth-Prakash-Resume.pdf`; the source keeps the short name on purpose.
- `resume-plain.tex` — backup single-column template. Not built by CI. Its content has NOT been
  kept in sync with `resume.tex` since the Sep 2026 rewrite; treat it as a layout fallback, not a
  current resume.
- `private.example.tex` — template for the git-ignored `private.tex` (phone number, work-permit
  line). Only read when `\privatebuild` is defined on the command line (see Commands). Never
  commit `private.tex` or `resume-private.pdf`: the repo and the release are public.
- `awesome-cv.cls` — vendored + locally patched Awesome-CV class (LPPL); `LICENCE-awesome-cv.txt`
  is its license. Credited in README, not in file names.
- `fonts/` — vendored Source Sans 3 OTF weights (SIL OFL, `fonts/LICENSE-SourceSans3.txt`),
  referenced by explicit path from `resume.tex`.
- `images/` — profile photo used in the header.
- `.github/workflows/build-and-sync.yml` — the only CI workflow; needs a `PORTFOLIO_PAT` repo
  secret to reach the sync step (see README's "GitHub Actions / Secrets setup"). Without it, the
  build/artifact/release steps still succeed — only the portfolio-sync step fails.

## Content rules (decided Sep 2026)

- Target role is **DevSecOps / Platform Engineer**; software-engineering work supports that story.
- Every metric and claim must trace to a source the user confirmed (older resumes, the portfolio
  repo's `data/en/sections/*.yaml`, GitHub READMEs, or an explicit answer). Do not add scale
  numbers, tool names, or outcomes that no source states — reviews and templates invent these.
- Phone number and work-permit status stay out of the public PDF (private build only).
- Languages: English (C1), German (A1, actively learning), Kannada (native). Permit: EU Blue Card.

## Page count

Two pages by design: page 1 = summary, skills, all experience; page 2 = projects, education,
certifications & awards, languages. A `cventry` is a `tabular*` and cannot break across pages, so
a longer entry can jump to page 2 and strand its section heading at the foot of page 1 — always
look at the page-1 render after editing anything near the page boundary (currently the first
Projects entry). A one-page variant was built and rejected (Sep 2026): it needed three projects and
the awards block cut.

## Local build

TeX Live 2026 is installed at `/usr/local/texlive/2026/bin/universal-darwin` (not on PATH — export
it, see Commands). Local output matches CI (same class, same vendored fonts, same FontAwesome
warning). Iterate locally and inspect with `pdfinfo`/`pdftotext`/`pdftoppm` (poppler); push only
when the render is approved, because every push to `resume.tex` updates the public release and
the live site. Then run `verify-resume-build` on the CI PDF.

## `awesome-cv.cls` gotchas (vendored, locally patched)

- **`\cventry{#1}{#2}{#3}{#4}{#5}`**: `#2` renders bold on line 1, `#1` renders small-caps on
  line 2 — the *opposite* of reading order. For Experience: `#1`=role, `#2`=company (company
  prominent). For Projects: `#1`=tech stack, `#2`=project name (name prominent) — easy to get
  backwards, verify against a render, not just the source.
- **`\XeTeXgenerateactualtext=1`** is set in `resume.tex` and must stay. Without it xdvipdfmx
  builds ToUnicode from the font cmap, and Source Sans 3 maps U+002D/U+2010/U+2011 to one glyph:
  every hyphen extracted as U+2011 (`cert‑manager`) and small caps extracted with dotless i
  (`ENGiNEER`). Verified with `pdftotext` on the CI PDF; with the flag both extract as ASCII.
- **FontAwesome6Brands icons have no ToUnicode/cmap** (`pdffonts` shows `uni=no` for that subfont
  only). `accsupp`'s `ActualText` mechanism (used by `\faAlt`) does not reliably attach to those
  glyphs via XeLaTeX's PDF backend — GitHub/LinkedIn icons rendered fine visually but were
  silently absent from `pdftotext` output. Fixed via a local `\faAltVisible` macro (always-visible
  plain text instead of relying on ActualText) — used only for GitHub/LinkedIn. `Renderer=HarfBuzz`
  disabling was tried first and was NOT the cause; left disabled anyway rather than re-testing an
  unrelated change against an already-verified document.
- **`~` in body text is a non-breaking space**, not a tilde. `~80\%` rendered as ` 80%`. Write
  `about 80\%` or `\textasciitilde`.
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
- **`cvhonors`** (3-column: date | title | location) looks visually disconnected next to a
  `cventries` table (2-column) in the *same* section. It is fine as its own section — that is how
  "Certifications & Awards" is done.
- ATS verification that actually matters: `pdftotext -raw file.pdf - | grep -oE 'punitpi|ppuneeth|cert-manager'`
  — a clean visual render does not prove the text layer is intact.

## CI trigger scope

`awesome-cv.cls` is in the path filter (`**/*.cls`), so *any* edit to it — including a
comment-only change — triggers a full rebuild + release + portfolio sync. `resume-plain.tex`,
README, and other non-listed files do not. This is intentional (a missed real change is worse than
an occasional harmless rebuild), but worth knowing before assuming a `.cls` comment tweak is free.

## GitHub Actions

- `xu-cheng/latex-action@v4`, `latexmk_use_xelatex: true`. Alpine-based image by default, no
  system fonts — `\setmainfont` on a named family (even TeX Gyre) risks a fontconfig lookup
  failure; either omit it (Latin Modern fallback) or vendor by file path.
- The compile step produces `resume.pdf`; a separate step renames it to
  `Puneeth-Prakash-Resume.pdf` before artifact/release/sync. A manual run with a different
  `root_file` will fail at that rename step.
- `softprops/action-gh-release@v3` on a fixed `tag_name` updates the existing release
  (`overwrite_files` defaults true) rather than erroring or duplicating — confirmed by pushing
  twice and checking the asset was replaced, not just the tag. It does NOT remove assets with a
  different name: after the Sep 2026 rename the old `resume-awesome.pdf` asset had to be deleted
  by hand (`gh release delete-asset latest resume-awesome.pdf`).
- Portfolio sync pushes with `PORTFOLIO_PAT` (fine-grained, `Contents: read/write`, scoped to
  `typedbyme` only) instead of the default `GITHUB_TOKEN` — PAT-authenticated pushes trigger
  downstream workflows; `GITHUB_TOKEN` pushes are suppressed to prevent recursive triggers. This
  is what makes the portfolio's Pages workflow fire automatically.
- Fine-grained PAT creation: the Permissions/Contents option only becomes selectable *after*
  choosing "Only select repositories" and picking the target repo — it's easy to miss if the repo
  picker step is skipped.
