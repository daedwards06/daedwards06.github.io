# Portfolio — Astro + GitHub Pages

Personal portfolio site for Dominique Edwards.

## Local Development

This repo requires Node.js 20+.

If you’re using the same setup as this workspace, Node 20 is provided via a conda env:

```powershell
conda activate portfolio-node
```

Then:

```powershell
npm ci
npm run dev
```

## Content

- Pages live in `src/pages/` (Home, Project, About, Contact).
- Projects are content-driven Markdown files in `src/content/projects/`.

To add a new project later:
1. Create a new `*.md` file in `src/content/projects/`.
2. Fill in the frontmatter (title, tags, links).
3. Keep `featured: false` until you want it to be the featured project.

## Build

```powershell
conda activate portfolio-node
npm run build
npm run preview
```

## Deploy (GitHub Pages)

- Deployment workflow: `.github/workflows/deploy.yml`
- Pushes to `main` build and deploy to GitHub Pages.
- The site is served from the root domain `https://daedwards06.github.io/` (`site` in `astro.config.mjs`, no `base`).

In GitHub repo settings, set Pages to deploy from **GitHub Actions**.
