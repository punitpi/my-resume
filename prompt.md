Act as a Principal DevOps and Software Engineer. Create a complete, production-ready GitHub repository setup for an ATS-friendly LaTeX resume with automated CI/CD deployment.

### Deliverables Required:

1. LaTeX Resume Template (resume.tex):
   - Clean, modern, single-page, ATS-compliant layout (Jake's Resume / ModernCV inspired).
   - Use XeLaTeX compatible packages (fontspec, geometry, hyperref, titlesec, enumitem).
   - Structured sections: Header (Name, Contact, LinkedIn, GitHub, Location), Technical Skills (Languages, Frameworks, Cloud/DevOps, Tools), Experience, Projects, and Education.
   - Tight, professional margins (0.5 - 0.75 in) with clean horizontal rules separating sections.

2. GitHub Actions CI/CD Pipeline (.github/workflows/build-and-sync.yml):
   - Trigger automatically on pushes to  involving .tex or .cls files.
   - Use  with XeLaTeX engine.
   - Generate  as a downloadable workflow artifact.
   - Auto-publish  as a GitHub Release under a rolling  tag.
   - Clone an external portfolio repository using a Personal Access Token (), copy  to , commit, and push changes back automatically.

3. Repository README.md:
   - Step-by-step setup instructions for local development (TeX Live / MacTeX / VS Code LaTeX Workshop).
   - Step-by-step GitHub Secrets configuration instructions for cross-repository sync.
   - Badge configuration for build status and direct PDF download link.

Output all files with full code blocks and exact folder paths.
