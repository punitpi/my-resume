---
name: add-resume-entry
description: Add a new job, project, education, certification, or award entry to resume.tex using the correct cventry argument order for that section, avoiding the bold/small-caps swap that's easy to get backwards.
disable-model-invocation: true
---

Adds an entry to `resume.tex`. This is a deliberate content edit — only run when the
user explicitly asks to add something, never proactively.

## The trap this skill exists to avoid

`\cventry{#1}{#2}{#3}{#4}{#5}` in `awesome-cv.cls` renders **`#2` bold on line 1** and **`#1`
small-caps on line 2** — the opposite of argument reading order. This got reversed once already
in this repo's Projects section (tech stack ended up bold-on-top instead of the project name) and
had to be fixed in a follow-up commit. Always check which field should be visually prominent
*before* writing the entry, not after rendering it.

## Templates by section

**Experience** (company prominent, role secondary — company is `#2`, role is `#1`):
```latex
  \cventry
    {<Role / Title>}
    {<Company>}
    {<Location>}
    {<Start> -- <End, or "Present">}
    {
      \begin{cvitems}
        \item {<achievement bullet, quantified where possible>}
      \end{cvitems}
    }
```

**Projects** (project name prominent, tech stack secondary — name is `#2`, tech stack is `#1`):
```latex
  \cventry
    {<Tech, Stack, Comma-Separated>}
    {<Project Name>}
    {}
    {<Year, or "Year -- Present">}
    {
      \begin{cvitems}
        \item {<one-line description, ideally fits without wrapping>}
      \end{cvitems}
    }
```

**Education** (institution prominent — institution is `#2`, degree is `#1`):
```latex
  \cventry
    {<Degree>}
    {<Institution>}
    {<Location>}
    {<Date Range>}
    {}
```

**Certifications & Awards** live in their own `cvhonors` block (date | title, issuer), not in
the Education `cventries` — a cert is not attached to a degree, and a 3-column `cvhonors` row
next to a 2-column `cventries` table in the *same* section looks disconnected (see CLAUDE.md):
```latex
  \cvhonor{<Certification or Award Name>}{<Issuer, or short context>}{}{<Year>}
```

## After adding an entry

1. Insert it inside the relevant `\begin{cventries} ... \end{cventries}` block in the right
   section, keeping entries in reverse-chronological order (most recent first) unless the user
   says otherwise.
2. Build locally (see CLAUDE.md "Local build") and check: still 2 pages with no section heading
   stranded at the bottom of page 1 (a `cventry` is a `tabular*` and cannot break across pages,
   so a longer entry can jump to page 2 and leave its heading behind), correct field prominence
   in the rendered preview, and ATS text extraction still includes all expected content. Then
   push and run `verify-resume-build` for the CI-built PDF.
