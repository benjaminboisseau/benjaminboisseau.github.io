# benjaminboisseau.github.io — blog

## Add a post

Drop a Markdown file in `content/posts/` with a little front matter:

```
+++
title = "My new post"
date = 2026-01-01
description = "One sentence shown on the home page."
+++

The body, in Markdown.
```

For the French version, add the same file name with a `.fr.md` extension
(`content/posts/my-new-post.fr.md`). That is the whole workflow.

```
zola serve     # live preview at http://127.0.0.1:1111
zola build     # output in public/
```

## Layout

```
config.toml              site config + the two languages
content/                 Markdown — the source of truth
  _index.*.md            home
  about.*.md             about page
  posts/*.md             articles (en) and *.fr.md (fr)
templates/               plain-HTML Tera templates (base, index, section, page)
static/style.css         the one stylesheet
.github/workflows/       builds + deploys to GitHub Pages on push
```

## Publishing

Currently a private repo. To go live: make it public, then Settings → Pages →
Source: **GitHub Actions**. The workflow in `.github/workflows/deploy.yml` builds
the site and deploys it on every push to `main`.
