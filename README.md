# spicyPoke.github.io

My Portfolio and Journal — a personal site built with [Hugo](https://gohugo.io/) and published
to GitHub Pages at [spicypoke.github.io](https://spicypoke.github.io/).

The site is hand-crafted: custom Go templates, plain CSS, and no third-party Hugo theme or
JavaScript framework.

## Tech Stack

- **Hugo** (extended edition, `>= 0.152`) — static site generator
- **Go templates** — all layouts in `layouts/`
- **Plain CSS** — `static/styles/style.css`, no preprocessor
- **Markdown + TOML front matter** — all content in `content/posts/`
- **GitHub Actions** — builds and deploys to GitHub Pages on every push to `main`

## Getting Started

### Prerequisites

- [Hugo extended](https://gohugo.io/installation/) `>= 0.152`
- Git (SSH access to GitHub if you plan to push)

### Local Development

```bash
git clone git@github.com:spicyPoke/spicyPoke.github.io.git
cd spicyPoke.github.io

hugo          # build the site into public/
hugo server   # run a local dev server (default: http://localhost:1313)
```

Run `hugo` after any change to verify the build is clean. `public/` is a build artifact and
is gitignored — never commit it.

## Project Structure

```
hugo.toml                   # site config (baseURL, taxonomies, pagination, date format)
archetypes/default.md       # front matter template for new posts
content/posts/              # posts as Markdown, one file each (kebab-case names)
layouts/                    # Go templates
  _default/                 #   baseof, home, single
  posts/                    #   /posts listing
  categories/  tags/        #   taxonomy listings
  partials/                 #   navbar, footer, list, leftBar, rightBar
static/
  styles/style.css          # all site styling
  images/                   # WebP images referenced by front matter
.github/workflows/hugo.yaml # CI: build + deploy to GitHub Pages
```

## Writing a Post

Scaffold a new post with:

```bash
hugo new content/posts/my-post.md
```

This generates front matter from `archetypes/default.md`:

```toml
date = '2026-08-15T12:00:00+07:00'   # RFC 3339 with local timezone offset
draft = true
title = 'My Post'
description = "One-sentence summary shown on the post header and in <meta>"
image = "/images/ai-slop.webp"       # thumbnail used in list pages
imageBig = "/images/ai-slop.webp"    # hero image on the post page
categories = [""]                    # e.g. "projects", "journal"
tags = [""]
authors = ["Experian"]
avatar = "/images/fubar.webp"
```

Guidelines:

- Set `draft = false` only when the post is ready to publish.
- Use kebab-case filenames (`home-automation.md`, not `Home Automation.md`).
- Images live in `static/images/` as WebP; reference them by absolute path (`/images/…`).
- Add posts under `content/posts/` only.
- Lists paginate at 10 posts per page (see `[pagination]` in `hugo.toml`).

## Styling & Layout

- All styling is in `static/styles/style.css`, organized under section comments
  (e.g. `/* HOMEPAGE START */`). Add new styles under the relevant section.
- Design tokens are CSS custom properties in `:root`: `--bg`, `--bgSoft`, `--text`,
  `--textSoft` (gold accent). Use them instead of hardcoding colors.
- Layouts reuse partials (`navbar`, `footer`, `list`, …). Prefer reusing or extending a
  partial over duplicating markup.
- Icons are inline SVGs in the templates (no icon font or image files).
- Fonts load from Google Fonts: Fira Sans (body), Dosis (headings).

## Deployment

Pushing to `main` triggers the workflow in `.github/workflows/hugo.yaml`, which:

1. Installs pinned toolchain versions (Hugo 0.152.2, Go 1.25.3, Node 22.20.0, Dart Sass 1.93.2).
2. Builds with `hugo --gc --minify`.
3. Uploads `public/` and deploys via `actions/deploy-pages`.

You can also trigger it manually from the Actions tab (`workflow_dispatch`).

## Contribution Guide

### Commit Messages

Follow [Conventional Commits](https://www.conventionalcommits.org/): `<type>: <summary>`,
all lowercase, present tense. Types used in this repo:

| Type   | When to use                              | Example                          |
| ------ | ---------------------------------------- | -------------------------------- |
| `feat` | New post, page, or user-visible feature  | `feat: add home automation post` |
| `docs` | Documentation changes                    | `docs: restore readme`           |
| `chore`| Maintenance, tooling, housekeeping       | `chore: remove example pics`     |

One commit per logical change. Don't bundle a new post together with unrelated refactors.

### Branching & Pull Requests

- Work on a short-lived branch (`git checkout -b post/my-post` or similar), then open a PR
  against `main`. Committing directly to `main` is fine for small, single-purpose changes.
- Keep PRs focused: one post or one layout/feature per PR. Renames or refactors of unrelated
  code belong in their own PR.
- Before opening a PR, run the verification checklist below.

### Style

- **Content**: follow the front matter conventions above; keep posts as pure Markdown
  (code blocks, lists, tables) — Hugo renders the rest.
- **Templates**: Go template style with consistent indentation (4 spaces). Reuse partials.
- **CSS**: semantic lowercase class names (`.listItem`, `.singleHead`, `.identityBar`),
  tokens from `:root`, no inline styles in templates.
- **Images**: WebP only, placed in `static/images/`; keep the existing naming style.

### Verification Checklist

Before pushing:

1. `hugo` builds without errors.
2. No new build warnings (e.g. missing layout for a new taxonomy).
3. Every referenced image exists under `static/images/` and the path matches exactly.
4. Front matter has all required fields; `draft = false` for published posts.
5. If templates/CSS changed, spot-check the result in `hugo server`.

### Known Limitations

- The `authors` taxonomy has no list layout yet, so `hugo` emits warnings about missing
  layouts for `term`/`taxonomy` kinds. A fix would mirror `categories/list.html` in a new
  `layouts/authors/list.html`.
