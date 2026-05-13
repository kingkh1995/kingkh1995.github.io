# AGENTS.md

**Personal tech blog** — Doraemon-themed SPA on GitHub Pages. Push `main` deploys.

## Structure

```
./index.html           # App: HTML + inline CSS + JS (~906 行)
./articles.json        # 文章清单 (id, title, category, file, date, summary)
./README.md            # 技术知识图谱 (分类目录权威来源)
./{category}/*.md      # Markdown 文章 (纯正文，无 front matter)
./assets/              # 图片、字体等静态资源
./.github/workflows/   # GitHub Pages 部署
```



## 关键映射

| 事项 | 位置 |
|------|------|
| 新增/修改文章 | `{category}/{id}.md` |
| 注册文章元信息 | `articles.json` (`date` 格式 `YYYY-MM-DD`) |
| 选择分类 | `README.md` 的 section header → 小写连字符目录名 |
| 新增分类映射 | `index.html` 中 `CATEGORY_NAMES` + `CATEGORY_ICONS`，同步更新 `README.md` |
| 修改样式 | `index.html` `<style>` (Doraemon CSS 变量) |
| 修改路由/渲染 | `index.html` `<script>` (hash SPA, marked.js) |

## 约定

- **文章标题层级**: 仅 h2–h4，无 h1/h5+。若源文含 h1（通常为文章标题），应直接删除该行，其内容作为 `articles.json` 中 `title` 字段的参考，而非降级为 h2
- **标点**: 中文全角 `（、，。）；` 英文术语两侧加空格
- **`articles.json`**: `file` 含 `.md` 后缀，`id` 为文件名（不含 `.md`），`summary` 用客观语气（介绍/说明/描述/概述）
- **原始文章处理**: 删除导航链接和标题引用块，图片路径改为 `assets/...`，移至对应分类目录

## 反模式

- 文章不用 h1/h5+，源文 h1 应删除（非降级为 h2）、不加 front matter
- `articles.json` 数据插入 innerHTML 必须用 `escapeHtml()`
- `#tailTop` 不用 `transform` 动画
- marked.js script 不加 `defer`

## 设计

Doraemon 蓝色系 (`--dora-blue*`)、黄色 (`--dora-yellow`)、红色 (`--dora-red`)、肚皮白 (`--dora-belly`)。标题用 ZCOOL KuaiLe 字体。响应式卡片网格：5 列 → 3 列 (768px) → 2 列 (480px)。

## 路由

Hash SPA: `#/` 首页 → `#/<category>` 文章列表 → `#/article/<category>/<id>` 文章详情。

## 本地开发

```bash
python -m http.server 8080
```

## 注意事项

- **marked.js 必须同步加载**（不加 `defer`）
- **Mermaid 图**懒加载 CDN，文章内容设置后调用 `renderMermaidBlocks()`
- **文章请求有 8s 超时**，`articleRequestId` 递增模式防止陈旧渲染
- **背景闪光** JS 注入，移动端 15 个，桌面 30 个，支持 `prefers-reduced-motion`
- **移动端溢出**: 宽表格自动 `.table-scroll`，长行内代码 `word-break: break-all`
- **性能**: 字体预加载、`appEl` DOM 缓存、`loading="lazy"`
- **`.gitignore`** 拦截: `/.idea/`, `/.superpowers/`, `/docs/`, `/.sisyphus/`, `/raw/`
