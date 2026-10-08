# 主题结构与加载机制

> 源码依据：`node_modules/hexo/dist/theme/{index,view}.js`、`dist/theme/processors/*`、`dist/hexo/load_config.js`、`dist/hexo/load_plugins.js`。核对版本：Hexo 7.3.0。

## 1. 主题如何被找到

- 主题名来自站点 `_config.yml` 的 `theme` 字段（`hexo/load_config.js`）。
- 解析顺序（先命中先用）：
  1. `<site>/themes/<theme>/`
  2. `<site>/node_modules/hexo-theme-<theme>/`（npm 安装的主题）
- 两者都不存在：`theme_dir` 仍指向 `themes/<theme>/`，主题 box 为空 → 所有页面拿不到布局（现象见 debugging.md）。
- 命中后 Hexo 重建主题 box（`ctx.theme = new Theme(ctx, {ignored})`），并忽略 `themes/*/node_modules`、`themes/*/.git`。
- 主题脚本目录固定为 `<theme>/scripts/`（`theme_script_dir`）。

## 2. 标准目录

```text
themes/<name>/
├── _config.yml     # 主题默认配置（优先级见 config-and-variables.md §1）
├── package.json    # 必须是合法 JSON；dependencies 里的 hexo-* 会被自动加载
├── layout/         # 模板（视图），扩展名 = 渲染器：.njk .pug .ejs .swg ...
├── source/         # 主题静态资源，生成时按原路径拷到站点根
├── languages/      # i18n 词条：zh-CN.yml / en.yml / default.yml ...
└── scripts/        # 主题脚本：注册 helper / tag / filter / generator / injector
```

## 3. 四个处理器（`dist/theme/processors/`）

| 文件 | 匹配模式 | 作用 |
|---|---|---|
| `config.js` | 主题根下 `_config.\w+` | 渲染结果**整体赋值**给 `theme.config`（不是合并） |
| `i18n.js` | `languages/*path` | 注册进 `theme.i18n`（hexo-i18n） |
| `source.js` | `source/**`（跳过隐藏文件、临时文件、含 `node_modules` 的路径） | 存入 `Asset` 模型 |
| `view.js` | `layout/*path` | 注册为视图 |

- `theme.config` 初始为 `{}`：主题不带 `_config.yml`，模板里 `theme.xxx` 全是 `undefined`。
- 视图注册：以「去掉扩展名的路径」为 key、按扩展名分桶。`theme.getView('index')` 返回该名字下第一个扩展名的视图，`theme.getView('index.njk')` 精确取；Windows 反斜杠会先归一化为 `/`。

## 4. source/ 的拷贝与同路径冲突

- `Asset` 记录路由路径 `path` 与真实文件 `source`；内置 `asset` generator 逐条写入路由。可被渲染的扩展名（如 `.yml`）先渲染再输出，其余原样流式拷贝。
- `router.set()` 是覆盖式写入：同一路由后写入者胜。
- **实测（Hexo 7.3.0，本机脚本复现 3 次）**：站点 `source/hello.txt` 与主题 `themes/<name>/source/hello.txt` 争 `hello.txt` 路由时，Asset 遍历顺序是站点在前、主题在后 → **主题文件胜出**。
- 结论：想覆盖主题的某个静态文件，**不要**依赖「站点 source 放同名文件」；改用不同路径 + 模板引用，或直接改主题文件。

## 5. scripts/ 与主题依赖（`hexo/load_plugins.js`）

- 启动时加载 `<theme>/scripts/` 与 `<site>/scripts/` 下所有 `.js`（主题脚本在前）。
- `package.json` 依赖扫描规则：
  - 站点：`dependencies` + `devDependencies`；主题：只读 `dependencies`。
  - 只加载名字匹配 `^hexo-` 或 `^@scope/hexo-` 的包；`hexo-theme-*` 永远不作为插件加载。
- **`package.json` 必须是合法 JSON**：写成 YAML 之类会让 `load_plugins` 里 `JSON.parse` 直接抛错，启动失败（实测踩过）。
- 惯例：`"name": "hexo-theme-<name>"`。

## 6. 新主题最小骨架 checklist

- [ ] `themes/<name>/package.json`（合法 JSON）
- [ ] `themes/<name>/_config.yml`（写全模板会读到的字段）
- [ ] `layout/index.<ext>`、`layout/post.<ext>`、`layout/page.<ext>`（`<ext>` 先确认渲染器已装）
- [ ] `layout/_partial/head.<ext>` 等片段，模板里用 `partial()` 引用
- [ ] `source/css/style.<ext>`，模板里用 `css('css/style.css')` / `js(...)` 引用
- [ ] `languages/zh-CN.yml`，模板里用 `__('key')` 取词
- [ ] 站点 `_config.yml` 设 `theme: <name>`（站点级覆盖写 `_config.<name>.yml`）
- [ ] `npx hexo clean && npx hexo generate --debug` 通过，`public/` 产物正确

## 7. 本仓库参照实现

- `themes/maupassant`：pug，当前启用，32 个布局，结构最简，适合当「最小参照」。
- `themes/butterfly`：pug，107 个布局，功能最全，查复杂交互/组件写法看它。
- `themes/shoka`：njk（27）+ ejs（3），带 `scripts/` 与 `example/`，查主题 scripts 用法看它。
