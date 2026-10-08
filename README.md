# hexo-theme-dev

面向 AI 编程代理（DeepSeek Harness / DSH）的 **Hexo 主题开发领域 Skill**：在开发、修改、调试自己的 [Hexo](https://hexo.io/) 主题时按需加载领域知识，把「凭记忆猜 API」变成「查过官方文档与源码才下结论」，提高主题开发的质量与效率。

## 为什么需要它

Hexo 的主题层有大量文档未覆盖的细节——主题配置优先级、模板变量的生成时机、布局回退链、`injector` 注入点、`partial` 与布局失败时的行为差异等，模型凭记忆回答很容易出错。本 Skill 的每条关键断言都回溯到 [Hexo 官方文档](https://hexo.io/docs/)或 Hexo 7.3.0 源码（`node_modules/hexo/dist/`），并在入口内置护栏：没有证据不下 API 结论、只改主题目录、改动后必须跑验证循环。

## 安装

方式一：手动（DSH 用户级技能目录）

```bash
git clone https://github.com/jiahui09/hexo-theme-dev.git
cp -r hexo-theme-dev/hexo-theme-dev ~/.agents/skills/
```

方式二：dotfiles 的 skill 管理（推荐，本仓库已纳入 `dsh/skills.lock`）

```bash
~/dotfiles/scripts/skill.sh add https://github.com/jiahui09/hexo-theme-dev.git   # 首次安装并写入 lock
~/dotfiles/scripts/skill.sh sync                                                # 换机时按 lock 补齐
```

## 最小使用示例

在会话里直接描述主题开发诉求即可自动命中，例如：

- 「我的 Hexo 主题配置改了不生效」
- 「给主题加一个注入点，把统计脚本塞进 head」
- 「主题生成的页面排版乱了，帮我排查」

也可以手动加载：在 DSH 中调用 `skill` 工具指定 `hexo-theme-dev`。

## 仓库结构

```text
hexo-theme-dev/
├── README.md
└── hexo-theme-dev/            # 技能目录（安装到 ~/.agents/skills/ 的就是它）
    ├── SKILL.md               # 入口：护栏 + 工作流 + 核心事实速览（保持精简）
    └── references/            # 按需加载的深度参考
        ├── theme-structure.md       # 主题目录与四类处理器解剖
        ├── config-and-variables.md  # 配置优先级、模板变量、页面字段
        ├── rendering.md             # 渲染管线 / 布局链 / injector / 扩展点
        ├── helpers.md               # helper 与内置 tag 清单
        └── debugging.md             # 命令速查、症状排查、验证清单
```

## 更新

1. 修改 `hexo-theme-dev/` 内的文件（SKILL.md 正文建议控制在 5000 字符内，细节进 `references/`）；
2. 提交并推送到 `main`；
3. 刷新安装与 lock：

```bash
~/dotfiles/scripts/skill.sh update hexo-theme-dev
~/dotfiles/scripts/skill.sh lint
```

## 贡献

个人技能仓库：欢迎通过 Issue 讨论、以 `feat/*` 分支提 PR 修改。

## License

暂未声明（对外复用前建议补充开源协议）。
