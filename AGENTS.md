# PROJECT KNOWLEDGE BASE

**Generated:** 2026-05-10
**Commit:** a1fdc8e
**Branch:** main

## OVERVIEW

Personal tech blog ("kaikoo的四次元口袋") published via GitHub Pages. Single-file SPA with no build step — push to `main` deploys automatically.

## STRUCTURE

```
./
├── index.html           # Entire app: HTML + inline CSS + inline JS (~834 lines)
├── articles.json        # Article manifest (id, title, category, file, date, summary)
├── techs.md             # Canonical category taxonomy — consult before adding articles
├── {category}/          # Markdown articles, one dir per tech category
│   └── *.md             # Article body only, no front matter
├── assets/              # Images, ZCOOLKuaiLe.ttf font, static files
└── .github/workflows/   # GitHub Pages deployment
```

Category dirs use English lowercase hyphenated names: `ai/`, `architecture/`, `jvm/`, `container/`, `dev-tools/`, etc. Some dirs exist preemptively without articles.

## WHERE TO LOOK

| Task | Location | Notes |
|------|----------|-------|
| Add/modify article content | `{category}/{article-id}.md` | No front matter; body only |
| Register article metadata | `articles.json` | Single source of truth for id, title, date, summary |
| Choose category for new article | `techs.md` | Section headers map to dir names |
| Add new category mapping | `index.html` → `CATEGORY_NAMES` + `CATEGORY_ICONS` | Also ensure `techs.md` covers it |
| Modify site styling/theme | `index.html` inline `<style>` | Doraemon-themed CSS custom properties |
| Modify routing/rendering | `index.html` inline `<script>` | Hash-based SPA, marked.js for markdown |
| Deploy | `.github/workflows/static.yml` | Auto-deploys on push to `main` |

## CONVENTIONS

### Category → Directory Mapping
- Section header in `techs.md` → English lowercase hyphenated dir name
- Examples: "AI & 新兴技术" → `ai/`, "容器 & 云原生" → `container/`, "开发工具" → `dev-tools/`

### Article Format
- Only h2–h4 headings (`##`, `###`, `####`); never h1 or h5+
- Chinese punctuation（：、，。）；English terms within Chinese prose get a space on each side
- Use `地` before verbs, `的` before nouns
- Use `、` for enumeration in Chinese; avoid `、和` — use `A、B 和 C` or `A、B、C`
- Technical converter/transformer terms: `转换器`, not `转化器`
- Pattern names: `Builder 模式`, not `build 模式`

### Incoming Articles (raw .md from root)
- Remove navigation links (`# [首页](/blog/)`) and redundant title blockquotes
- Fix image paths to be relative from root (`assets/...` not `/blog/assets/...`)
- Move to correct category dir, delete original from root

## ANTI-PATTERNS (THIS PROJECT)

- **Never use h1 or h5+ in articles** — breaks heading hierarchy
- **Never add front matter to .md files** — metadata lives only in `articles.json`
- **Never insert `articles.json` data into innerHTML without `escapeHtml()`** — XSS risk
- **Never add `transform` animations to `#tailTop`** — conflicts with `bottom`/`opacity` transitions
- **Do not commit `/docs/` without `-f`** — blocked by `.gitignore`

## DESIGN SYSTEM

Doraemon-themed CSS custom properties in `:root`:
- `--dora-blue` / `--dora-blue-dark` / `--dora-blue-deep` / `--dora-blue-light` — blue body tones
- `--dora-yellow` / `--dora-yellow-glow` — bell gold
- `--dora-red` / `--dora-red-dark` — collar/nose/tail
- `--dora-belly` (#FFFEF9) — content area off-white

Font: ZCOOL KuaiLe (self-hosted `assets/ZCOOLKuaiLe.ttf`) for titles, categories, article h1. Body uses system sans-serif stack.

Responsive card grid: 5-col → 3-col at 768px → 2-col at 480px.

## ROUTING

Hash-based SPA with three patterns:
- `#/` → home: category card grid (categories with articles only)
- `#/<category>` → category page: article list sorted by date desc
- `#/article/<category>/<id>` → article page: fetches .md, renders via marked.js

## COMMANDS

```bash
# Local dev
python -m http.server 8080
# Test at http://localhost:8080/#/
```

## NOTES

- Windows Git Bash (msys2): LF→CRLF warnings on commit are harmless
- Mobile overflow: wide tables auto-wrapped in `.table-scroll`; long inline code needs `word-break: break-all`; blockquotes need `overflow-wrap: break-word`
- No build step, no dependencies beyond CDN-loaded `marked.js`
- **marked.js must load synchronously** — do NOT add `defer`; the inline script checks `typeof marked` immediately on execution
- Background sparkles are JS-injected (not static HTML) with mobile-adaptive count (15 vs 30) and `prefers-reduced-motion` support
- Performance optimizations in place: font preload, `appEl` DOM cache, fetch timeout via `AbortController`, `loading="lazy"` on article images
