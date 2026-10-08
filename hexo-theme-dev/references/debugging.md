# 调试与验证

> 源码依据：`dist/hexo/index.js`、`dist/hexo/router.js`、`dist/plugins/console/*`、`dist/hexo/load_plugins.js`。核对版本：Hexo 7.3.0。

## 1. 常用命令

```bash
npx hexo clean                        # 删 public/ 与 db.json（改动不生效时的第一武器）
npx hexo generate --debug             # 全量生成 + 全部详细日志
npx hexo generate --cache             # 路由缓存（增量生成；排查时别用）
npx hexo server                       # 本地预览；--debug 同样可用；-p 3000 换端口
npx hexo list route                   # 列出路由，核对产物路径
npx hexo list post|page|category|tag   # 核对解析出的内容
npx hexo config                       # 查看合并后的最终配置
npx hexo render <file>                # 单文件跑渲染器，隔离验证模板/Markdown
npx hexo version                      # 版本与运行环境
```

- `--debug` 是全局 flag（minimist 解析，任意位置均可），置 `env.debug = true`。
- 模板里调试：`{{ log(page) }}`、`{{ inspect(obj) }}`（debug helper，输出到日志）。

## 2. 症状 → 排查表

| 症状 | 依次检查 |
|---|---|
| 改动完全不生效 | ① `hexo clean`（db.json + public 缓存）② 是否看错配置文件（优先级：`theme_config` > `_config.<theme>.yml` > 主题 `_config.yml`）③ 改的文件是否真被加载（主题名对不对、路径对不对）④ `hexo server` 是否在跑旧进程 |
| 页面「没有外壳」，只有内容/半截 HTML | **布局静默丢失**：`_resolveLayout` 找不到不报错。核对 front-matter `layout` 名、视图是否存在于 `<theme>/layout/`、扩展名是否有渲染器、路径是相对还是绝对（相对路径以当前视图目录为基准） |
| `Partial xxx does not exist. (in ...)` | partial 路径按当前视图目录解析；核对文件名与大小写；片段必须在 `<theme>/layout/` 下 |
| 模板里 `theme.xxx` 是 undefined | 主题没放 `_config.yml`，或字段被更高优先级配置覆盖/合并时丢字段；用 `npx hexo config` 与 `{{ inspect(theme) }}` 核对 |
| 页面 404 / 资源 404 | `npx hexo list route` 看路由；`config.root` 是否与部署路径一致（子路径部署必须设 `root: /blog/`）；资源引用是否漏了 `url_for` / `css()` / `js()` |
| 样式/脚本原样漏出或 404 | 渲染器缺失（`.styl`/`.sass` 要装对应 renderer）；或 `source/` 文件没被拷贝（被隐藏文件/临时文件规则跳过） |
| 主题 scripts 里的注册没生效 | 文件是否在 `<theme>/scripts/` 且为 `.js`；语法错误会让 `loadPlugin` 报 `Plugin load failed`（看 `--debug` 日志） |
| 启动即崩 `JSON.parse ... is not valid JSON` | 某个 `package.json` 写坏了（主题的也算）；必须是合法 JSON |
| injector 注入没出现 | entry 写错会**静默落到 `head_end`**；`to` 类型与页面类型不匹配（见 rendering.md §4）；页面是否走了主题渲染（`layout: false` 的文章不执行 injector） |
| 代码高亮/Markdown 异常 | 站点 Markdown 渲染器与 `highlight` 配置；不是主题问题 |

## 3. 核心机制与常见坑

### 缓存

- **`db.json`**：文章内容、资源索引都缓存在内；`before_exit` 时 `save_database` 回写。改了 front-matter/资源没反应 → `hexo clean`。
- **路由缓存**：`hexo generate --cache`，或站点配置 `server.cache: true`（`hexo server` 的 watch 重渲染才复用已渲染路由；默认关闭）开启。连续排查「怪异的不生效」时先 `hexo clean` 再 `hexo generate` 全量。
- **`fragment_cache` / `partial({cache:true})`**：单次进程内缓存，跨命令不保留。

### 路径

- 所有输出到 HTML 的地址都应走 `url_for` / `css` / `js`（吃 `config.root`）；`full_url_for` 吃 `config.url`（SEO/OG 用）。裸写 `/css/x.css` 在子路径部署下会 404。
- 视图内部路径（partial、`_resolveLayout`）用 `/`，Windows 下 Hexo 会归一化，但自写脚本要自己处理。
- `theme/source/` 与 `site/source/` 同路径冲突时**主题胜出**（实测，见 theme-structure.md §4）。

### 运行时

- 主题 box 与配置只在启动时装配：改 `_config.yml` 的 `theme` 字段需要重启 `hexo server`。
- `hexo server` 对 box 变更的响应有约 100ms 防抖；连续快速改动可能只触发最后一次。
- 插件/脚本加载失败不会中止整个进程（`load_plugins` 记 error 继续），**必须看日志**才能发现。

## 4. 验证清单（每次改动后）

1. `npx hexo clean && npx hexo generate --debug` 无 ERROR/WARN（新增的更要查）。
2. `npx hexo list route` 或 `ls -R public/` 核对预期产物路径。
3. `grep` 关键产物：新组件/新字段确实出现在 `public/**/*.html`。
4. `npx hexo server` 肉眼核验至少两类页面（首页 + 文章页）；改动涉及全局时抽查归档/分类/标签页。
5. 涉及移动端/暗色模式/动画的改动，浏览器 DevTools 里按实际设备核对（不要只看桌面截图）。

## 5. 性能

- 大站生成慢：确认没有在模板里做重复重活（同一片段用 `partial({cache:true})`）；`--debug` 日志里逐路由的 `Rendering HTML` 耗时可定位慢页面。
- `hexo server` 增量渲染依赖路由缓存；改动不生效与「缓存」永远优先怀疑。
- 代码高亮默认在渲染期完成；换 `highlight.js` 配置属站点配置，不在主题侧解决。
