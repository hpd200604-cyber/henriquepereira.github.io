### Option 1: Conventional Commits

```text
feat: initialize shader portfolio for Henrique Pereira and configure GitHub Pages deployment

- Clean up upstream template bloat by removing tools, docs, husky, devcontainer, linters, and npm build configs
- Restructure project to run on pure Jekyll and Ruby without Node dependencies
- Configure portfolio metadata, branding, and author details for Henrique Pereira in _config.yml and _data/authors.yml
- Prune 31 unused locale files, retaining only pt-BR and en, and disable PWA service worker caching
- Fix favicon, avatar, and webmanifest asset references to resolve 404 errors
- Add reusable _includes/hlsl-viewer.html component to run native Unity HLSL code live in WebGL with custom aspect ratios
- Add introductory unlit shader post and a full breakdown post for Henrique's soap bubble shader with real-time 3D simulation
- Update _tabs/about.md with a Technical Artist / Shader Developer profile template
- Create .github/workflows/pages-deploy.yml for automated deployment to GitHub Pages via GitHub Actions
```

### Option 2: Simple and direct

```text
Initialize Henrique Pereira shader portfolio and deployment workflow

Cleaned up upstream Chirpy repository files, adapted the project for pure Jekyll/Ruby, configured branding and pages for Henrique Pereira, added an interactive HLSL-to-WebGL shader runner with sample posts including the soap bubble shader breakdown, and added the GitHub Pages deployment workflow.
```
