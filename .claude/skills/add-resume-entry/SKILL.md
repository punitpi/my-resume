---
name: add-resume-entry
description: Add a new job, project, or education/certification entry to resume-awesome.tex using the correct cventry argument order for that section, avoiding the bold/small-caps swap that's easy to get backwards.
disable-model-invocation: true
---

Adds an entry to `resume-awesome.tex`. This is a deliberate content edit — only run when the
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

**Education / Certification** (institution/issuer prominent — issuer is `#2`, credential is `#1`):
```latex
  \cventry
    {<Degree or Certification Name>}
    {<Institution or Issuing Body>}
    {<Location, or blank for a cert>}
    {<Date or Date Range>}
    {}
```
Give a certification its own `\cventry` in the same `cventries` block as education — don't nest
it as a `cvitems` bullet under the degree entry, which visually attaches it to that specific
degree and is misleading for an unrelated professional credential (hit this exact issue before).

## After adding an entry

1. Insert it inside the relevant `\begin{cventries} ... \end{cventries}` block in the right
   section, keeping entries in reverse-chronological order (most recent first) unless the user
   says otherwise.
2. Run the `verify-resume-build` skill to confirm: still 1 page (a new entry may push it to 2 —
   see CLAUDE.md's page-fit trim order: tighten spacing first, cut content last, and ask the user
   before dropping anything), correct field prominence in the rendered preview, and ATS text
   extraction still includes all expected content.
