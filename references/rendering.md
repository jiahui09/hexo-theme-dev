# 渲染管线与扩展点

> 源码依据：`dist/hexo/index.js`（`createLoadThemeRoute`、`_routerRefresh`、`_generateLocals`）、`dist/theme/view.js`、`dist/extend/{filter,injector,tag,helper}.js`、`dist/hexo/render.js`、`dist/plugins/*`。核对版本：Hexo 7.3.0。

## 1. 端到端管线

```text
启动： update_package → load_config → load_theme_config → load_plugins（加载插件 + 站点/主题 scripts/）
     ↓
准备： source box + theme box 处理文件 → 数据模型（Post/Page/Asset/词条/视图）
     ↓
before_generate filter（内置：render_post 渲染文章内容）
     ↓
generators 跑一遍 → 每条结果 { path, layout[], data } 进路由
     ↓
template_locals filter（内置：注入 __ / _p，规范化 page.lang / canonical_path）
     ↓
视图渲染 view.render：
   ① 视图 frontmatter → locals（去掉 layout/_content）
   ② 按扩展名找渲染器编译执行（helpers 已绑定进 locals）
   ③ 执行 after_render:<输出类型> filter（每个视图每次渲染都跑一次）
   ④ frontmatter 有 layout → 找父布局递归渲染，把子结果作为 body 传入（layout: false 断链）
     ↓
injector.exec（对整页 HTML 注入 head_begin/head_end/body_begin/body_end）
     ↓
after_render:html（内部别名 _after_html_render，与 ③ 是同一个 filter 仓库）
     ↓
路由输出 → hexo server 或写入 public/
```

**要点：`after_render:html` 可能被执行多次**（每个 html 视图渲染后、整页渲染后、`render.render` 渲染任意 html 输出后，走的都是同一个 `_after_html_render` 仓库，见 `dist/extend/filter.js` 的 `typeAlias`）。自定义这类 filter 必须**幂等**。

## 2. 视图与布局链（`dist/theme/view.js`）

- 视图源文件 = `<theme>/layout/<path>`；frontmatter 的 `layout` 决定父布局。
- 布局解析 `_resolveLayout(name)`：先按**相对当前视图的路径**找（`dirname(view) + name`），找不到再按绝对路径找；找到但就是自己则跳过。
- 父布局拿到 `body`（子输出）+ `layout: false`（防死循环）。
- **找不到布局不报错**：`_resolveLayout` 返回空时直接返回当前结果 → 页面「没有外壳」渲染出来。这是高频静默故障，见 debugging.md。
- `layout: false` 或生成器判定不走布局时，路由数据就是纯内容（文章的 `post.content`）。

## 3. 生成器的布局回退链

| 来源 | layout 数组 |
|---|---|
| post generator | front-matter `layout`（若非 `post` 则前置）→ `post → page → index` |
| page generator | front-matter `layout`（若非 `page` 则前置）→ `page → post → index` |
| hexo-generator-index | `index → archive` |
| hexo-generator-archive | `archive → index` |
| hexo-generator-category | `category → archive → index` |
| hexo-generator-tag | `tag → archive → index`（`/tags/` 总览页 `tag-index → tag → archive → index`） |

按顺序找第一个存在的视图渲染（`createLoadThemeRoute`）。

## 4. injector API（`dist/extend/injector.js`）

```js
// 注册（主题 scripts/ 里）
hexo.extend.injector.register(entry, value, to);
```

- `entry`：`head_begin` / `head_end` / `body_begin` / `body_end`。**写错的 entry 会静默落到 `head_end`**（源码 `this.store[entry] || this.store.head_end`）。
- `value`：字符串，或函数（注册时立即执行取返回值）。
- `to`：目标页面类型，默认 `'default'`；可选 `home` / `post` / `page` / `archive` / `category` / `tag` / 具体布局名。解析顺序（`_getPageType`）：`__index`→home、`__post`→post、`__page`→page、`archive`→archive、`category`→category、`tag`→tag、最后回退 `page.layout`。
- 注入实现：整页 HTML 上做字符串替换，输出带 `<!-- hexo injector <entry> start/end -->` 标记；同一 entry 重复注入会被标记拦住（防重入）。
- 生效范围：只对经 `createLoadThemeRoute` 的主题 HTML 路由生效；纯内容路由（`layout: false` 的文章）不会执行 injector。

## 5. 扩展点一览（`dist/extend/`）

| API | 注册内容 | 常用场景 |
|---|---|---|
| `hexo.extend.helper.register(name, fn)` | 模板函数，`this` = locals | 主题工具函数 |
| `hexo.extend.tag.register(name, fn, options)` | `{% name %}` 标签（Nunjucks 扩展；`options` 可为 `true`=块标签、`{ async: true }`） | 自定义标签 |
| `hexo.extend.filter.register(type, fn, priority)` | 管线钩子，`priority` 默认 10、小者先跑 | 内容/HTML 改写 |
| `hexo.extend.generator.register(name, fn)` | 生成路由（`fn(locals)` 返回 `{path, layout, data}`） | 自定义页面 |
| `hexo.extend.injector.register(entry, value, to)` | 见 §4 | 注入统计/脚本 |
| `hexo.extend.processor.register(pattern, fn)` | 处理 source 文件 | 自定义文件类型 |
| `hexo.extend.renderer.register(ext, output, fn, sync)` | 渲染器 | 支持新模板语言 |
| `hexo.extend.console.register(...)` | 自定义子命令 | 主题工具命令 |
| `hexo.extend.deployer.register` / `highlight` | 部署 / 代码高亮配置 | 少用 |

filter 类型清单（内置注册见 `dist/plugins/filter/index.js`）：

- `before_post_render`（内置：反引号代码块、titlecase）
- `after_post_render`（内置：external_link、excerpt）
- `before_generate`（内置：render_post）/ `after_generate`
- `before_exit`（内置：save_database 存 db.json）
- `template_locals`（内置：i18n）
- `after_render:html`（别名 `_after_html_render`；内置：external_link、meta_generator）、`after_render:css` / `after_render:js` 等按输出类型
- `_after_html_render`（新名称，与上者同仓库）
- `new_post_path` / `post_permalink`（内置：permalink 规则）
- `after_init`（init 序列末尾执行）、`after_generate`（每次生成结束后执行）

## 6. 渲染器

- **Hexo 内置**（`dist/plugins/renderer/index.js`）：`njk`/`j2`（Nunjucks）、`html`/`htm`、`css`、`js`、`json`、`yml`/`yaml`（→json）。
- **Markdown 不是内置的**，必须装 `hexo-renderer-*`（本博客用 `@upupming/hexo-renderer-markdown-it-plus`）。
- 其余模板语言靠站点安装：`hexo-renderer-ejs`、`hexo-renderer-pug`、`hexo-renderer-swig`、`hexo-renderer-stylus`、`hexo-renderer-sass` 等（本博客都已装）。装没装看站点 `package.json` 的 `hexo-renderer-*`。
- 规则：**layout 文件的扩展名必须有对应渲染器**，否则该文件不会被当成视图渲染（模板源码原样漏出或根本进不了视图表）。

## 7. 主题 scripts/ 最小示例

```js
// themes/<name>/scripts/extend.js —— 启动时自动执行
hexo.extend.helper.register('my_sum', (a, b) => a + b);          // 模板: {{ my_sum(1, 2) }}

hexo.extend.injector.register('body_end', '<script src="/js/x.js"></script>', 'post');

hexo.extend.filter.register('after_render:html', html =>          // 必须幂等
  html.replace('cdn.example.com', 'cdn.proxy.com'));

hexo.extend.tag.register('hello', (args, content) => `Hello ${args[0]}`);  // {% hello world %}
```

官方文档：<https://hexo.io/docs/plugins>（writing a plugin）、<https://hexo.io/docs/tag-plugins>。
