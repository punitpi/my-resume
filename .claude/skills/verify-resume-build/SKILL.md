---
name: verify-resume-build
description: Watch the latest resume CI run to completion, download the built PDF, and check page count, ATS text extraction, and render a preview. Use after any push to resume.tex or awesome-cv.cls in this repo.
---

Iterate locally first (TeX Live 2026 is installed, see CLAUDE.md "Local build") — the CI PDF is
the one that ships, so this skill verifies it after the push. Every push to `resume.tex` also
updates the public release and the live site, so push only when the local render is approved.

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
   gh run download <run-id> --repo punitpi/my-resume --name resume-pdf --dir /tmp/resume-check
   ```
   (A manual `workflow_dispatch` run with `root_file: resume-plain.tex` uploads no artifact — the
   rename step expects `resume.pdf`. Build the plain template locally instead.)

4. **Check page count** (the primary resume is 2 pages by design — see CLAUDE.md "Page count"):
   ```bash
   pdfinfo /tmp/resume-check/Puneeth-Prakash-Resume.pdf | grep Pages
   ```
   If it grew to 3, or page 1 ends with a stranded section heading, see CLAUDE.md's page-fit
   gotchas before making spacing changes (the `acvSectionContentTopSkip` trap especially — going
   below ~2mm can overlap text instead of tightening it).

5. **Check ATS text extraction** — a clean visual render does NOT prove the text layer is
   intact (this bit us once with GitHub/LinkedIn icons that rendered fine but were silently
   missing from extracted text):
   ```bash
   pdftotext -raw /tmp/resume-check/Puneeth-Prakash-Resume.pdf - | grep -oE 'punitpi|ppuneeth|cert-manager'
   ```
   `punitpi` (GitHub), `ppuneeth` (LinkedIn) and `cert-manager` (ASCII hyphen, not U+2011 — see
   the `\XeTeXgenerateactualtext` note in CLAUDE.md) must all appear. If any is missing, the text
   layer has regressed — don't ship it.

6. **Render a preview and look at it** (poppler must be installed — `brew install poppler` if
   `pdftoppm` isn't found):
   ```bash
   pdftoppm -png -r 150 /tmp/resume-check/Puneeth-Prakash-Resume.pdf /tmp/resume-check/preview
   ```
   Then read `/tmp/resume-check/preview-1.png` and `preview-2.png` with the Read tool to visually
   confirm no overlapping text, correct field order, no stranded heading at the foot of page 1,
   and no obviously broken layout.

## If verifying multiple hypotheses

Per CLAUDE.md: batch independent changes into one push rather than testing one variable at a
time — each round-trip through this loop costs ~2 minutes. If two theories about a bug are both
plausible and cheap to apply together, apply both in one commit and let this skill's checks tell
you whether either (or neither) worked.

## Release / live-site checks (only after `PORTFOLIO_PAT` sync succeeds)

To confirm the change actually reached the public release and the live portfolio site:
```bash
curl -sL -o /tmp/release-check.pdf -w "HTTP:%{http_code} size:%{size_download}\n" \
  "https://github.com/punitpi/my-resume/releases/latest/download/Puneeth-Prakash-Resume.pdf"
pdfinfo /tmp/release-check.pdf | grep Pages

curl -sL -o /tmp/live-site-check.pdf -w "HTTP:%{http_code} size:%{size_download}\n" \
  "https://typedbyme.puneeth.io/files/Resume.pdf"
pdftotext -raw /tmp/live-site-check.pdf - | grep -oE 'punitpi|ppuneeth'
```
