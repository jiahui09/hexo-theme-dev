---
name: hexo-theme-dev
description: 开发、修改或调试 Hexo 主题时使用。触发场景：新建自己的 Hexo 主题、修改 layout 模板与样式、查模板变量/helper/tag/partial/injector、主题配置不生效、主题报错、生成产物异常、主题脚手架。基于 Hexo 7 官方文档与本地源码给出主题结构、配置优先级、渲染管线、内置能力清单与调试验证流程。
---

# Hexo 主题开发（hexo-theme-dev）

面向「开发自己的 Hexo 主题」的领域技能。本文件只放工作流与索引，细节按需读 `references/`，不要一次性全读。

## 护栏

1. **不凭记忆断言 API**：任何关于变量/helper/行为的说法，都要能指回 `node_modules/hexo/dist/` 下的源码文件或官方文档（hexo.io/docs/*）。拿不准就先读源码再下结论。
2. **动主题前先确认三件事**：当前 `theme` 值；哪个配置文件在生效（优先级见下）；layout 文件扩展名对应哪个渲染器（没有渲染器 = 模板原样输出或直接报错）。
3. 只改 `themes/<name>/` 与主题相关配置；要动站点配置、文章内容或部署前，先说明影响。
4. 每次改动都走「验证循环」，未验证不交付。

## 工作流

### 1. 定位现状

```bash
grep -E "^theme:|^theme_config" _config.yml   # 当前主题与站点级覆盖
ls themes/                                     # 本地主题
ls _config.*.yml                               # 是否存在 _config.<theme>.yml
grep hexo-renderer package.json                # 已装渲染器（njk/html/js/css/yml 内置）
```

参照实现（本仓库）：`themes/maupassant`（pug，当前启用，最简）、`themes/butterfly`（pug，107 个布局，功能全）、`themes/shoka`（njk + ejs，带主题 scripts）。

### 2. 按需读参考

| 要做的事 | 读 |
|---|---|
| 建主题骨架、放静态资源、写主题 scripts/、主题依赖 | `references/theme-structure.md` |
| 改主题配置、查模板变量 / page 字段 / i18n | `references/config-and-variables.md` |
| 模板渲染流程、layout 继承、injector、扩展点 API | `references/rendering.md` |
| 找 helper / 内置 tag 的用法 | `references/helpers.md` |
| 报错、改动不生效、产物异常、性能 | `references/debugging.md` |

### 3. 验证循环

```bash
npx hexo clean && npx hexo generate --debug    # 全量重生成 + 完整日志
npx hexo server                                # 本地预览；--debug 看详细日志
```

- 日志/产物报错 → 按 `references/debugging.md` 对症排查 → 修复 → 复验。
- 改动「不生效」的前三嫌疑人固定为：缓存（先 `hexo clean`）→ 配置优先级（看错了文件）→ 渲染器缺失（该扩展名没人渲染）。

## 核心事实速览

- **配置优先级**：站点 `_config.yml` 的 `theme_config` > 站点根 `_config.<theme>.yml` > 主题自带 `_config.yml`。
- **模板 locals**：`config`（站点配置）、`theme`（主题配置）、`page`、`site`、`path`、`url`（绝对 URL）、`layout`、`env`、`view_dir`、`__` / `_p`（i18n），所有 helper 也绑定为 locals。
- **布局链**：视图 frontmatter 的 `layout` 指向父布局，父布局用 `body` 接收子输出；文章回退链 `post → page → index`；`layout: false` 直接输出原始内容。
- **partial**：按相对当前视图的路径解析；找不到**直接抛错**；永远不套布局。
- **injector**：`head_begin / head_end / body_begin / body_end` 四个注入点，按页面类型（home/post/page/archive/category/tag/布局名）选择注入，只对走主题渲染的 HTML 路由生效。
- **同路径冲突**：站点 `source/` 与主题 `source/` 同名文件争同一路由，实测（Hexo 7.3.0）主题侧胜出——不要指望「往站点 source 丢同名文件覆盖主题」。
