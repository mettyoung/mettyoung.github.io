# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Personal blog/portfolio site (mettyoung.com) built on Jekyll, hosted via GitHub Pages (uses the `github-pages` gem to stay in lockstep with GitHub's supported plugin/theme versions — do not add plugins outside what `github-pages` bundles, or GitHub Pages deploy will silently diverge from local builds).

## Commands

```bash
bundle install          # install gems (Ruby version pinned in .ruby-version)
bundle exec jekyll serve # local dev server with live rebuild, http://localhost:4000
bundle exec jekyll build # build static site into _site/
bundle update github-pages # bump to latest GitHub-Pages-supported gem versions
```

No test suite, linter, or CI workflow exists in this repo (`.github/` only contains `FUNDING.yml`).

## Architecture

- `_config.yml` — site-wide settings (name, description, analytics IDs, footer social links, plugin list). `include: ['_pages']` is required since Jekyll excludes underscore-prefixed dirs by default — pages living in `_pages/` need this to be picked up.
- `_layouts/default.html` — outer HTML shell (head, masthead/nav, footer with social icons + copyright). All other layouts extend this via `layout: default` front matter.
- `_layouts/page.html` / `_layouts/post.html` — wrap `default` for static pages vs. blog posts. `post.html` adds author/date, category tags, share links, and Disqus comments (`_includes/disqus.html`).
- `_pages/` — static pages (About, Projects, Categories, Search, Archive). Each needs a `permalink:` in front matter since these aren't in the default Jekyll `pages` scan path.
- `_posts/` — blog posts, standard Jekyll `YYYY-MM-DD-title.markdown` naming. Front matter `categories:` drives grouping on `/categories/`. Posts can be marked `published: false` to keep as drafts.
- `_includes/` — reusable snippets: `meta.html` (SEO/meta tags), `analytics.html`/`analytics_head.html` (GA4 via `google_analytics_ga4` config), `svg-icons.html` (footer social icons), `pullquote.html`, `disqus.html`.
- `_sass/` + `assets/style.scss` — styling. `style.scss` is the Sass entry point Jekyll compiles to `assets/style.css`; partials (`_variables`, `_reset`, `_custom-style`, `_highlights`, `_darcula`, `_svg-icons`) are imported from there.
- Search (`/search/`) is client-side via `assets/simple-jekyll-search.min.js` against the generated `search.json` (index of all posts) — no backend/server search.
- `assets/i-openapi.yaml` + `i-openapi.html` — a standalone OpenAPI spec/viewer embedded as a static asset for a specific blog post series, unrelated to the rest of the site.

## Workflow

- When asked to change an existing article (`_posts/`), commit and push to `origin master` after making the edit — no need to ask first. GitHub Pages auto-builds from master, so this publishes the change live.
- After pushing, verify the change live (curl or WebFetch the post URL, e.g. `mettyoung.com/<slug>/`, checking for the new content/markup) before reporting done. GitHub Pages builds take a bit — poll rather than checking once immediately.
