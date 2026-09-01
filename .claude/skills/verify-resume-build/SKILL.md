---
name: verify-resume-build
description: Watch the latest resume CI run to completion, download the built PDF, and check page count, ATS text extraction, and render a preview. Use after any push to resume-awesome.tex, resume.tex, or awesome-cv.cls in this repo.
---

This machine has no local `xelatex`/`pdflatex`/`latexmk` and no Docker/Podman (see CLAUDE.md).
Every `.tex`/`.cls` change is verified by pushing and inspecting the CI-built PDF — this skill
runs that loop.

## Steps

1. **Identify the run.** After a push, get the most recent run:
   ```bash
   gh run list --repo punitpi/my-resume --limit 1
   ```
   Note the run ID (the numeric field near the end of the row).

2. **Watch it to completion:**
   ```bash
   gh run watch <run-id> --repo punitpi/my-resume --exit-status
   ```
   A non-zero exit only means *something* in the job failed — check *which* step. If only
   "Sync PDF into portfolio" failed and everything before it (compile, artifact, release) is
   green, that's the expected/known failure mode when `PORTFOLIO_PAT` isn't set — not a build
   problem. Anything else failing (especially "Compile ... with XeLaTeX") is a real problem.

3. **Download the built PDF** into a fresh temp dir (don't reuse a stale one from an earlier
   check in the same session):
   ```bash
   rm -rf /tmp/resume-check && mkdir -p /tmp/resume-check
   gh run download <run-id> --repo punitpi/my-resume --name resume-awesome-pdf --dir /tmp/resume-check
   ```
   Use `--name resume-pdf` instead if verifying `resume.tex` via a manual `workflow_dispatch` run.

4. **Check page count** (the primary resume must stay at 1 page):
   ```bash
   pdfinfo /tmp/resume-check/resume-awesome.pdf | grep Pages
   ```
   If more than 1 page, see CLAUDE.md's page-fit gotchas before making further spacing changes
   (the `acvSectionContentTopSkip` trap especially — going below ~2mm can overlap text instead
   of tightening it).

5. **Check ATS text extraction** — a clean visual render does NOT prove the text layer is
   intact (this bit us once with GitHub/LinkedIn icons that rendered fine but were silently
   missing from extracted text):
   ```bash
   pdftotext -raw /tmp/resume-check/resume-awesome.pdf - | grep -oE 'punitpi|ppuneeth'
   ```
   Both `punitpi` (GitHub) and `ppuneeth` (LinkedIn) must appear. If either is missing, the text
   layer has regressed — don't ship it.

6. **Render a preview and look at it** (poppler must be installed — `brew install poppler` if
   `pdftoppm` isn't found):
   ```bash
   pdftoppm -png -r 150 /tmp/resume-check/resume-awesome.pdf /tmp/resume-check/preview
   ```
   Then read `/tmp/resume-check/preview-1.png` (and `-2.png` etc. if multi-page) with the Read
   tool to visually confirm no overlapping text, correct field order, and no obviously broken
   layout.

## If verifying multiple hypotheses

Per CLAUDE.md: batch independent changes into one push rather than testing one variable at a
time — each round-trip through this loop costs ~2 minutes. If two theories about a bug are both
plausible and cheap to apply together, apply both in one commit and let this skill's checks tell
you whether either (or neither) worked.

## Release / live-site checks (only after `PORTFOLIO_PAT` sync succeeds)

To confirm the change actually reached the public release and the live portfolio site:
```bash
curl -sL -o /tmp/release-check.pdf -w "HTTP:%{http_code} size:%{size_download}\n" \
  "https://github.com/punitpi/my-resume/releases/latest/download/resume-awesome.pdf"
pdfinfo /tmp/release-check.pdf | grep Pages

curl -sL -o /tmp/live-site-check.pdf -w "HTTP:%{http_code} size:%{size_download}\n" \
  "https://typedbyme.puneeth.io/files/Resume.pdf"
pdftotext -raw /tmp/live-site-check.pdf - | grep -oE 'punitpi|ppuneeth'
```
