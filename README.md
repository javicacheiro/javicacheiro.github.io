# javicacheiro.com

Personal site of Javier Cacheiro López ([javicacheiro](https://github.com/javicacheiro)) —
Scientific Director of the [OneHealth DataSpace](https://dataspace.cesga.es) at
[CESGA](https://www.cesga.es).

Built with [Astro](https://astro.build) and
[Astro Theme Pure](https://github.com/cworld1/astro-theme-pure) (see `LICENSE-theme`).
Deployed to GitHub Pages via GitHub Actions (`.github/workflows/deploy.yml`).

## Development

```bash
npm install
npm run dev      # local dev server
npm run build    # production build to dist/
npm run preview  # preview the production build
```

## Writing a post

Add a folder under `src/content/blog/<slug>/` with an `index.md`:

```markdown
---
title: 'Post title'
description: 'Short description (max 160 chars).'
publishDate: 2026-07-19
tags:
  - hpc
---

Content here.
```

## Notes

- `public/` is copied verbatim to the site root: it holds the `CNAME` for the custom
  domain and the legacy presentation folders (`infoday/`, `workshop/`, `workshopml/`).
- Dark theme is the default for first-time visitors (seeded in `src/layouts/BaseLayout.astro`).
- Site configuration lives in `src/site.config.ts` and `astro.config.ts`.
