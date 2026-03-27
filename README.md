# Blogo

This repository is a Jekyll blog deployed to GitHub Pages with GitHub Actions and styled with the Minimal Mistakes theme.

## Publishing a post

1. Create a Markdown file in `_posts/` named `YYYY-MM-DD-title.md`.
2. Add front matter like this:

```md
---
title: My post title
date: 2026-03-27 09:00:00 +0200
---
```

3. Commit and push to `main`.
4. GitHub Actions builds the site and deploys it to GitHub Pages.

## Theme notes

The site uses Minimal Mistakes as a `remote_theme`, pinned in `_config.yml`.
If you want to change the theme version or skin, update `remote_theme` and `minimal_mistakes_skin`.

## One-time GitHub setting

In your repository settings, go to `Settings > Pages` and set `Source` to `GitHub Actions`.

If you rename the repository, update `baseurl` in `_config.yml` to match the new repository name.
