# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Personal tech blog ("kaikoo的四次元口袋") published via GitHub Pages. Single-file SPA with no build step — push to `main` deploys automatically.

## Architecture

- `index.html` — the entire app: HTML structure + inline CSS + inline JS. All rendering, routing, and styling in one file.
- `articles.json` — article manifest. Each entry: `id`, `title`, `category` (English dir name), `file` (relative path to .md), `date` (ISO), `summary`.
- `*.md` — articles with YAML front matter (`---` delimited title/date/category/summary), stored in English-named category dirs (`architecture/`, `java/`, etc.).
- `assets/` — images and static files.
- `techs.md` — personal category taxonomy. Consult this file when choosing or creating a category for an article; it defines the canonical category hierarchy.

## Routing

Hash-based SPA with three route patterns:
- `#/` → home: category card grid (only categories with articles shown)
- `#/<category>` → category page: article list sorted by date desc
- `#/article/<category>/<id>` → article page: fetches .md, parses front matter, renders via marked.js

Category dir names (English) and Chinese display names are mapped in `CATEGORY_NAMES` constant; category emoji icons in `CATEGORY_ICONS`.

## Design System

Doraemon-themed CSS custom properties in `:root`:
- `--dora-blue` / `--dora-blue-dark` / `--dora-blue-deep` / `--dora-blue-light` — blue body tones
- `--dora-yellow` / `--dora-yellow-glow` — bell gold
- `--dora-red` / `--dora-red-dark` — collar/nose/tail
- `--dora-belly` (#FFFEF9) — content area off-white

Font: ZCOOL KuaiLe (self-hosted via `assets/ZCOOLKuaiLe.ttf` + `@font-face`) used for site title, subtitle, category names, article titles, and article h1. Body text uses system sans-serif stack. Responsive: 3-col card grid → 2-col at 768px → 1-col at 480px.

## Local Dev

```
python -m http.server 8080
```

Test at `http://localhost:8080/#/`. All files are static — no build, no dependencies.

## Deployment

GitHub Actions workflow in `.github/workflows/static.yml` deploys entire repo to GitHub Pages on push to `main`. Just commit and push.

## Adding Content

1. Determine the article's category by consulting `techs.md` — the canonical category taxonomy. Use the closest matching category (the section header in `techs.md` maps to a directory name)
2. Write `.md` article with YAML front matter in the appropriate category dir
3. Add entry to `articles.json` with matching `category`, `file`, and `id`
4. If new category, add mapping to `CATEGORY_NAMES` and icon to `CATEGORY_ICONS` in `index.html`. Also ensure `techs.md` covers this technology area
5. If new category dir, create the directory
6. Check and fix issues before finalizing: punctuation, typos, grammar, terminology, markdown format consistency, and text polishing — without adding new content

### Incoming Articles

Raw `.md` files may appear in the repo root. Before integrating:
- Remove navigation links (`# [首页](/blog/)`) and redundant title blockquotes
- Fix image paths to be relative from root (e.g., `assets/...` not `/blog/assets/...`)
- Move to the correct category dir, delete the original from root

### Article Formatting Rules

- Only use h2–h4 headings (`##`, `###`, `####`); never h1 or h5+
- Chinese text uses Chinese punctuation（：、，。）；English terms within Chinese prose get a space on each side
- Use `地` before verbs (尽可能地), `的` before nouns (精准定义的)
- Use `、` for enumeration in Chinese, not `，` between list items
- `、和` is redundant — use `A、B 和 C` or `A、B、C`
- Technical converter/transformer terms use `转换器`, not `转化器`
- Pattern name: `Builder 模式` not `build 模式`

## Gotchas

- Windows Git Bash (msys2) — LF→CRLF warnings on commit are harmless
- `.gitignore` blocks `/docs/` — force-add with `-f` if docs changes must be committed
- `escapeHtml()` must be used on all data from `articles.json` before inserting into innerHTML
- The tail back-to-top button uses `bottom`/`opacity` transitions — do not add `transform` animations to it
