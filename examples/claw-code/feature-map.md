# 功能清单：Claw Code

> 生成时间：2026-06-16 | 代码版本：`d229a9b` | 分析工具：code-to-req v2
> 提取来源：CLI subcommands (main.rs:2804)、Python subparsers (main.py:23-74)、commands_snapshot.json (141 命令)、tools_snapshot.json (94 工具)

---

## 功能域：对话与会话

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 多轮对话 | 与 AI 进行连续多轮交互，保持上下文 | runtime crate / runtime.py turn-loop |
| 2 | 会话恢复 | 从之前中断的对话继续工作 | CLI `resume` subcommand / session_store.py |
| 3 | 会话导出 | 将对话记录导出为可分享的格式 | CLI `export` subcommand |
| 4 | 会话压缩 | 长对话自动压缩上下文，节省 token | `/compact` command |
| 5 | 会话分支 | 创建会话分支，探索不同方向 | `/branch` command |

## 功能域：代码编辑

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 文件读取 | AI 读取项目文件理解代码 | FileReadTool (tools crate) |
| 2 | 文件编辑 | AI 直接修改代码文件 | FileEditTool (tools crate) |
| 3 | 文件创建 | AI 创建新文件 | FileWriteTool (tools crate) |
| 4 | 代码搜索 | 按模式/内容搜索代码 | GrepTool / GlobTool (tools crate) |
| 5 | Diff 查看 | 查看当前改动的差异 | CLI `diff` subcommand |

## 功能域：终端与执行

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | Bash 执行 | AI 执行 shell 命令完成任务 | BashTool (tools crate / runtime bash.rs) |
| 2 | 沙箱模式 | 隔离执行环境，防止危险操作 | CLI `sandbox` / sandbox-toggle command |
| 3 | 权限控制 | 工具执行前需用户授权 | permissions.py / runtime 权限系统 |

## 功能域：项目理解

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 目录添加 | 将额外目录加入分析范围 | `/add-dir` command |
| 2 | 上下文管理 | 查看/调整 AI 的项目上下文 | `/context` command |
| 3 | 文件列表 | 查看当前工作涉及的文件 | `/files` command |
| 4 | LSP 集成 | 利用语言服务获取类型/定义信息 | LSPTool (tools crate) |

## 功能域：任务规划

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 计划模式 | 先规划再执行，避免盲目修改 | EnterPlanModeTool / `/plan` command |
| 2 | 任务管理 | 创建/追踪/完成子任务 | TaskCreateTool / TaskUpdateTool / `/tasks` |
| 3 | 工作树隔离 | 在独立分支上做实验性修改 | EnterWorktreeTool / ExitWorktreeTool |
## 功能域：Agent 与多模型

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 子 Agent 派发 | 将子任务分派给独立 Agent 并行处理 | AgentTool / `/agents` command |
| 2 | 模型切换 | 根据任务选择不同 AI 模型 | CLI `model` subcommand / `/model` |
| 3 | 技能系统 | 加载专用 Skill 扩展 AI 能力 | SkillTool / `/skills` command |
| 4 | 远程模式 | 在远程服务器上执行任务 | remote_runtime.py / CLI remote 相关 |

## 功能域：插件与扩展

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 插件安装 | 从市场安装第三方插件 | `/plugin` + `/install` commands |
| 2 | 插件市场 | 浏览可用插件 | CLI `marketplace` / BrowseMarketplace |
| 3 | MCP 服务 | 连接外部工具服务（数据库/API 等） | MCPTool / `/mcp` command |
| 4 | Hooks 系统 | 用户自定义事件钩子 | `/hooks` command / plugins crate |

## 功能域：Git 与协作

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 提交代码 | AI 辅助生成 commit message 并提交 | `/commit` command |
| 2 | 创建 PR | 提交+推送+创建 Pull Request 一步完成 | `/commit-push-pr` command |
| 3 | 代码审查 | AI 审查代码变更 | `/review` command |
| 4 | Issue 处理 | 读取 GitHub Issue 并处理 | `/issue` command |
| 5 | PR 评论 | 处理 PR 上的评论反馈 | `/pr_comments` command |

## 功能域：开发者体验

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 主题配色 | 自定义终端配色方案 | `/theme` / `/color` commands |
| 2 | Vim 模式 | 使用 Vim 快捷键编辑 | `/vim` command |
| 3 | 状态栏 | 实时显示工作状态 | `/statusline` command |
| 4 | 快捷键 | 查看/配置键位绑定 | `/keybindings` command |
| 5 | 费用追踪 | 查看 API 使用量和费用 | `/cost` / `/usage` commands |
| 6 | 诊断检查 | 检测环境配置问题 | CLI `doctor` subcommand |

## 功能域：Web 与信息获取

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 网页抓取 | 获取指定 URL 的内容 | WebFetchTool (tools crate) |
| 2 | 网页搜索 | 搜索互联网获取最新信息 | WebSearchTool (tools crate) |
| 3 | 图片处理 | 理解用户上传的截图/图片 | imageProcessor (tools crate) |

## 功能域：定时与自动化

| # | 功能 | 用户价值 | 实现位置 |
|---|------|---------|---------|
| 1 | 定时任务 | 创建周期性自动执行的任务 | CronCreateTool / CronDeleteTool |
| 2 | 记忆系统 | 跨会话记住用户偏好和项目信息 | `/memory` command / agentMemory |

---

## 统计

| 指标 | 数值 |
|------|------|
| 功能域 | 10 个 |
| 用户可用功能 | 42 个 |
| 底层命令注册数 | 141 个 |
| 底层工具注册数 | 94 个 |

> 注：141 个命令中包含大量内部组件（UI 步骤、辅助模块），面向用户的顶层功能约 42 个。
