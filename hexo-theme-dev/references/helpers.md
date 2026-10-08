# 内置 helper 与 tag 清单

> 源码依据：`dist/plugins/helper/index.js`、`dist/plugins/tag/index.js`（逐个注册名核对，Hexo 7.3.0）。参数签名以官方文档为准：<https://hexo.io/docs/helpers>、<https://hexo.io/docs/tag-plugins>。
>
> 通用事实：helper 在模板里以 locals 函数调用（`{{ url_for(...) }}` / `<%- partial(...) %>`，随模板语言变化）；helper 内部 `this` = 当前 locals（可访问 `this.page`、`this.config`、`this.theme`）。

## 1. helper（按用途分组）

### URL 与路径

| 名字 | 用途 |
|---|---|
| `url_for(path)` | 拼上 `root` 的站点内路径（`/js/a.js`） |
| `full_url_for(path)` | 拼上 `config.url` 的绝对 URL |
| `relative_url(from, to)` | 站内相对路径 |
| `link_to(url, text, options)` | 生成 `<a>` |
| `mail_to(mail, text, options)` | 生成 mailto 链接 |

### 页面类型判断

`is_home()`、`is_home_first_page()`、`is_post()`、`is_page()`、`is_archive()`、`is_year()`、`is_month()`、`is_category()`、`is_tag()`、`is_current(path, equal)` —— 全部基于 `page` 上的 `__index/__post/__page/archive/category/tag` 标记与当前路径，返回布尔。

### 列表与分页

| 名字 | 用途 |
|---|---|
| `list_archives(options)` | 归档列表 |
| `list_categories(options)` | 分类列表 |
| `list_tags(options)` | 标签列表 |
| `list_posts(options)` | 文章列表 |
| `paginator(options)` | 翻页条（用 `page.current/total` 等字段） |
| `tagcloud(options)` / `tag_cloud(options)` | 标签云（两者同一实现） |

### 日期时间

`date(date, format)`、`date_xml(date)`、`time(date)`、`full_date(date, format)`、`relative_date(date, from)`、`time_tag(date, format)`（输出 `<time>`）、`moment(date)`（Moment.js 实例）。

### 文本处理

`escape_html(str)`、`strip_html(str)`、`truncate(str, options)`、`word_wrap(str, width)`、`titlecase(str)`、`trim(str)`、`number_format(num)`、`markdown(text)`（渲染 Markdown）、`render(text, options)`（按引擎渲染）、`inspect(obj)`（调试打印对象）、`log(...)`（写 Hexo 日志）。

### 资源与元信息

| 名字 | 用途 |
|---|---|
| `css(path)` / `js(path)` | 生成 `<link>` / `<script>` 标签（走 `url_for`，可接数组） |
| `favicon_tag(href)`、`feed_tag(href, options)`、`image_tag(src, options)` | 对应 `<link>`/`<img>` 标签 |
| `gravatar(email, size)` | 头像 URL |
| `open_graph(options)` | Open Graph `<meta>` 组 |
| `meta_generator()` | `<meta name="generator">` |
| `toc(str, options)` | 从正文生成目录 |
| `search_form(options)` | 搜索表单 |

### 缓存与杂项

- `fragment_cache(key, fn)`：片段缓存（同一生成进程内）。
- `partial(name, locals, options)`：引入子视图，见下。

**`partial` 语义（`dist/plugins/helper/partial.js`，重要）**：

- 路径按「当前视图所在目录 + name」解析，失败再按布局根绝对路径解析；**找不到抛 `Partial xxx does not exist.`**。
- 默认继承当前 locals（`this`），`options.only: true` 时只带传入的 `locals`。
- 强制 `layout: false`（片段永不套布局）。
- `options.cache: true | 'key'` → 走 `fragment_cache`，重复片段只渲染一次。

## 2. 内置 tag（Markdown 里的 `{% ... %}`）

执行环境是 Hexo 自带的 Nunjucks（`dist/extend/tag.js`），`{% raw %}` 可用；官方文档 <https://hexo.io/docs/tag-plugins>。

| 标签 | 用途 | 备注 |
|---|---|---|
| `quote` / `blockquote` | 引用块 | 块级 |
| `code` / `codeblock` | 代码块 | 块级 |
| `pullquote` | 浮动引文 | 块级 |
| `iframe` | 内嵌 iframe | |
| `img` / `image` | 图片（source 路径） | |
| `include_code` / `include-code` | 引入代码文件 | 异步 |
| `a` / `link` / `anchor` | 链接 | |
| `post_path` | 取文章路由路径 | |
| `post_link` | 链接到文章（标题作锚文本） | |
| `asset_path` | 取文章资源路径 | |
| `asset_link` | 链接文章资源 | |
| `asset_img` / `asset_image` | 显示文章资源图片 | |
| `url_for` / `full_url_for` | 模板外取 URL | |

说明：

- 同一行的别名（如 `img`/`image`）是同一实现的两个注册名。
- 站点还装了 `hexo-tag-aplayer`、markdown-it 插件（emoji、pangu）等第三方 tag/扩展，用法见各自文档。
- 内容里想原样输出 `{% %}`：包在 `{% raw %} ... {% endraw %}` 里。

## 3. 找不到时怎么办

1. 先确认它是不是**主题/插件自己注册的**（在 `themes/<name>/scripts/`、站点 `scripts/`、或 `hexo-*` 插件包里 `grep "extend.helper.register"` / `extend.tag.register`）。
2. 再查官方 helper/tag 文档。
3. 都没有 → 属于臆造 API，不要在模板里用。
