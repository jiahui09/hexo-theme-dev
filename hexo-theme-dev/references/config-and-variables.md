# 配置优先级与模板变量

> 源码依据：`dist/hexo/load_theme_config.js`、`dist/hexo/index.js`（`mergeCtxThemeConfig`、`_generateLocals`）、`dist/hexo/locals.js`、`dist/plugins/filter/template_locals/i18n.js`、各 generator 包源码。核对版本：Hexo 7.3.0。

## 1. 配置优先级（谁覆盖谁）

由低到高：

1. 主题自带 `themes/<name>/_config.yml` → 由 `theme/processors/config.js` 读入 `hexo.theme.config`
2. 站点根 `_config.<theme>.yml` → `hexo/load_theme_config.js` 读入 `config.theme_config`（与已有 `theme_config` 深合并，`theme_config` 字段优先）
3. 站点 `_config.yml` 里的 `theme_config:` 段 → 同一函数内压过第 2 层

最后 `mergeCtxThemeConfig`（`dist/hexo/index.js`）把 `config.theme_config` 深合并进 `theme.config`。

**结论：站点 `theme_config` > `_config.<theme>.yml` > 主题 `_config.yml`**（与官方文档一致）。

- 主题模板里的 `theme` 变量 = 合并后的 `hexo.theme.config`；`config` = 站点配置。
- 站点级定制主题时优先写 `_config.<theme>.yml`，别改主题文件，升级主题不丢配置。
- 改了配置不生效 → 见 debugging.md（优先级 + 缓存）。

## 2. 模板 locals（每个模板里可直接用的变量）

由 `_generateLocals`（`hexo/index.js`）+ `template_locals` filter（i18n）+ 视图 frontmatter 汇成：

| 变量 | 含义 |
|---|---|
| `config` | 站点 `_config.yml`（合并后） |
| `theme` | 主题配置（按 §1 合并后） |
| `page` | 当前页面数据（见 §3） |
| `site` | 站点集合：`posts` / `pages` / `categories` / `tags` / `data` |
| `path` | 当前路由相对路径（如 `2024/01/01/hello/`） |
| `url` | 当前页面绝对 URL（基于 `config.url`） |
| `layout` | 固定字符串 `'layout'`（别当布尔判断用） |
| `env` | `{ version, cmd, debug, silent, init, env, safe, args }` |
| `view_dir` | `<theme>/layout/` 的绝对路径 |
| `__` / `_p` | i18n 取词（`_p` 处理复数） |
| 全部 helper | 以函数形式绑定进 locals，模板里直接调用 |

- 视图自身 frontmatter 的字段也会并入 locals（`layout`、`_content` 除外）——即布局文件里可以直接用 `title` 这类 front-matter 字段。
- helper 内部的 `this` 就是这套 locals（所以 helper 里能 `this.page`、`this.config`）。

## 3. `page` 字段（按页面类型）

**文章页**（post generator，`layout: post`）——post 模型字段：`title`、`date`、`updated`、`content`、`excerpt`、`more`、`tags`、`categories`、`path`、`permalink`、`source`、`slug`、`layout`；生成器附加 `prev` / `next` 和 `__post = true`。

- front-matter `layout: false`（或 `'false'`）→ 不走主题，直接把 `post.content` 输出为该路由。
- 摘要：正文里 `<!-- more -->` 之前是 `excerpt`，之后是 `more`（`after_post_render` 的 excerpt filter，匹配 `<!-- ?more ?-->`）；front-matter 已写 `excerpt:` 时 `more` = 全文。`<!-- more -->` 处会替换为 `<span id="more"></span>`。

**独立页**（page generator）：同上字段 + `__page = true`；`layout: false`/`'off'` 不走主题；布局回退链 `page → post → index`。

**首页/列表**（hexo-generator-index）：`__index = true` + 分页字段：`base`、`total`、`current`、`current_url`、`posts`（当前页文章）、`prev`/`prev_link`、`next`/`next_link`（hexo-pagination）。布局回退链 `index → archive`。

**归档**（hexo-generator-archive）：分页字段 + `archive = true`；年/月/日归档另有 `year`、`month`、`day`。布局链 `archive → index`。

**分类**（hexo-generator-category）：分页字段 + `category = <名>`。布局链 `category → archive → index`。

**标签**（hexo-generator-tag）：分页字段 + `tag = <名>`。布局链 `tag → archive → index`；`tag_generator.enable_index_page: true` 时额外生成 `/tags/` 总览页，布局链 `tag-index → tag → archive → index`，并带 `tags` 集合。

**`site` 集合**（`hexo/locals.js`）：`site.posts`、`site.pages`、`site.categories`、`site.tags`、`site.data`（站点 `source/_data/*.yml` 转成对象）。

`is_*` helper 依赖这些标记：`is_home()` = `page.__index`、`is_home_first_page()` = `__index && current === 1`、`is_post()` = `__post`、`is_page()` = `__page`、`is_archive()` = `page.archive` 等。

## 4. i18n

- 词条文件：`<theme>/languages/<lang>.yml`（`theme/processors/i18n.js`）。
- 语言判定顺序（`template_locals/i18n.js`）：
  1. front-matter 的 `page.lang` 或 `page.language`
  2. 用 `config.i18n_dir`（默认 `':lang'`）匹配 `locals.path` 得到的语言（如 `/en/xxx/` 里的 `en`），命中则同时设置 `page.canonical_path`
  3. `config.language`（可为数组）的第一个
- 回退链：`[当前语言, ...config.language, ...已加载语言, 'default']`。
- 模板用法：`__('key')`、`_p('key', n)`；`page.lang` 已被规范化，可直接输出到 `<html lang>`。

## 5. 与主题相关的常用站点配置

`url`（绝对地址，决定 `url_for` 输出）、`root`（默认 `/`，所有静态资源路径前缀）、`language`、`permalink`、`per_page`、`pagination_dir`、`archive_dir`、`theme`、`theme_config`、`i18n_dir`、`skip_render`、`ignore`。官方文档：<https://hexo.io/docs/configuration>、<https://hexo.io/docs/variables>、<https://hexo.io/docs/internationalization>。
