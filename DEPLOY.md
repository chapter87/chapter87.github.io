# Deploying Chapter87

The site auto-deploys to **GitHub Pages** on every push to `main`.

- Repo: https://github.com/chapter87/chapter87.github.io
- Live URL: https://chapter87.github.io
- Deploy config: `.github/workflows/deploy.yml` (GitHub Actions → Pages)
- Site URL is set in `astro.config.mjs` (`site:`).

## Everyday workflow

```bash
cd ~/chapter87
npm run dev          # preview locally at http://localhost:4321
# ...write a post in src/content/blog/*.md ...
git add -A && git commit -m "new post: <title>"
git push             # GitHub Actions rebuilds + deploys automatically
```

## Writing a post

Drop a `.md` file in `src/content/blog/`. Frontmatter:

```md
---
title: 'Your title'
description: 'One-line summary shown on the homepage and in search results.'
pubDate: 'Aug 20 2026'
heroImage: '../../assets/blog-placeholder-1.jpg'
---

Body in markdown.
```

## Custom domain (when you buy chapter87.com)

1. In the repo: Settings → Pages → Custom domain → `chapter87.com`.
2. At your registrar, add DNS records pointing to GitHub Pages:
   - `A` records for the apex to `185.199.108.153`, `.109.153`, `.110.153`, `.111.153`
   - `CNAME` for `www` → `chapter87.github.io`
3. Update `site:` in `astro.config.mjs` to `https://chapter87.com` and push.
