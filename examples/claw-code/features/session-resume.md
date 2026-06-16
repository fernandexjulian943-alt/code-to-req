# 会话恢复 — 需求细节

> 生成时间：2026-06-16 | 代码版本：`d229a9b` | 基于 feature-map「对话与会话 > 会话恢复」
> 验证状态：已通过（步骤 3.5 行号/计数/字段批量验证）

## 功能概述

- **所属功能域**：对话与会话
- **用户价值**：从之前中断的对话继续工作，保持上下文连续性
- **当前状态**：已实现（CLI `--resume` 参数 + REPL `/resume` 命令 + 自动会话持久化）

## 用户故事

| # | 角色 | 行为 | 目的 |
|---|------|------|------|
| US-1 | 作为开发者 | 我想恢复最近一次的对话 | 以便从中断处继续工作，无需重新描述上下文 |
| US-2 | 作为开发者 | 我想恢复指定的历史会话 | 以便回到某个特定工作状态继续 |
| US-3 | 作为开发者 | 我想在恢复时执行 slash 命令 | 以便快速查看恢复会话的状态（如 /status /compact） |
| US-4 | 作为开发者 | 我想跨工作区恢复会话 | 以便在不同项目间切换时找回之前的对话 |

## 验收标准

| # | 场景 | Given | When | Then |
|---|------|-------|------|------|
| AC-1 | 恢复最近会话 | 存在至少一个有消息的会话 | 执行 `claw --resume latest` | 加载最近的非空会话，显示文件路径+消息数+轮次数 |
| AC-2 | 恢复指定会话 | 知道 session-id 或文件路径 | 执行 `claw --resume <id>` | 加载指定会话 |
| AC-3 | 别名支持 | - | 使用 "latest" / "last" / "recent" | 均解析为最近非空会话 |
| AC-4 | 恢复+执行命令 | - | 执行 `claw --resume latest /compact` | 先恢复会话再执行 /compact |
| AC-5 | REPL 内恢复 | 在 REPL 交互中 | 输入 `/resume <session-path>` | 当前会话切换为目标会话，显示恢复报告 |
| AC-6 | 无会话可恢复 | 无任何历史会话 | 执行 `claw --resume latest` | 报错提示无可恢复会话 |
| AC-7 | 所有会话为空 | 有会话但消息数=0 | 执行 `claw --resume latest` | 报错区分"全为空"和"无会话" |
| AC-8 | 排除当前会话 | REPL 中当前会话为 A | `/resume latest` | 跳过 A，恢复次新的会话 |
| AC-9 | 跨工作区恢复 | 用别名引用 | `--resume latest` 目标在不同 workspace | 允许恢复，打印 note 提示来源 workspace |
| AC-10 | 不支持的命令 | - | `claw --resume session.jsonl /commit` | 报错"not yet implemented"，exit code 2 |

## 数据模型

从代码中提取的核心结构：

```rust
// runtime/src/session.rs:117-133
pub struct Session {
    pub version: u32,
    pub session_id: String,
    pub created_at_ms: u64,
    pub updated_at_ms: u64,
    pub messages: Vec<ConversationMessage>,
    pub compaction: Option<SessionCompaction>,
    pub fork: Option<SessionFork>,
    pub workspace_root: Option<PathBuf>,
    pub prompt_history: Vec<SessionPromptEntry>,
    pub last_health_check_ms: Option<u64>,
    pub model: Option<String>,
    persistence: Option<SessionPersistence>,  // 私有字段
}
```

```rust
// runtime/src/session_control.rs:20-26
pub struct SessionStore {
    sessions_root: PathBuf,    // e.g. <cwd>/.claw/sessions/<workspace_hash>/
    workspace_root: PathBuf,   // 规范化的工作区路径
}

// runtime/src/session_control.rs:536-539
pub struct SessionHandle {
    pub id: String,
    pub path: PathBuf,
}

// runtime/src/session_control.rs:542-551
pub struct ManagedSessionSummary {
    pub id: String,
    pub path: PathBuf,
    pub created_at_ms: u64,
    pub updated_at_ms: u64,
    pub modified_epoch_millis: u128,
    pub message_count: usize,
    pub parent_session_id: Option<String>,
    pub branch_name: Option<String>,
}
```

**存储路径规则**：
- 主格式：`.claw/sessions/<workspace_fingerprint>/<session_id>.jsonl`
- 遗留格式：`.claw/sessions/<workspace_fingerprint>/<session_id>.json`
- workspace_fingerprint：FNV-1a 哈希（`session_control.rs:512-520`），对 workspace_root 的 lossy UTF-8 做 64 位哈希取 16 位 hex
- 常量：`PRIMARY_SESSION_EXTENSION = "jsonl"` / `LEGACY_SESSION_EXTENSION = "json"`（`session_control.rs:529-530`）
- 别名：`"latest"` / `"last"` / `"recent"` 均解析为最近非空会话（`session_control.rs:531-533`）

## API 接口

| 入口 | 触发方式 | 参数 | 行为 |
|------|---------|------|------|
| CLI 参数 | `claw --resume <ref> [/cmd...]` | session-path / session-id / alias | 加载会话 + 可选执行命令 |
| Slash 命令 | `/resume <session-path>` | 必填 session-path | REPL 内切换当前会话 |

**CLI 参数解析**（`main.rs:3338-3344`）：
```rust
fn parse_resume_args(args: &[String], ...) -> Result<CliAction, String> {
    // 无参数 → 默认 "latest"
    // 首参看起来像 slash command → session=latest, 参数作为命令
    // 否则 → 首参=session_path, 剩余=命令
}
```

**REPL 内恢复**（`main.rs:8480-8517`）：
- 调用 `load_session_reference_excluding` 加载（排除当前 session）
- 用加载的 session 构建新 runtime
- 替换当前 runtime + session handle
- 输出恢复报告（文件路径、消息数、轮次数）

**恢复报告格式**（`main.rs:6175-6181`）：
```
Session resumed
  Session file     {path}
  Messages         {count}
  Turns            {turns}
```

## 业务规则 & 边界条件

- **会话引用解析优先级**（`session_control.rs:102-136`）：
  1. 是别名（latest/last/recent）→ 找最近非空会话（排除当前 ID）
  2. 是绝对路径或含扩展名 → 直接加载文件
  3. 否则 → 尝试在 sessions_root 下按 ID 查找（先 .jsonl 后 .json）
  4. 兜底查 legacy_sessions_root

- **"最近非空"逻辑**（`session_control.rs:179-212`）：
  1. 先在当前 workspace 的 session 命名空间查找（`message_count > 0` 且 `id != exclude`）
  2. 找不到 → 扫全局命名空间 `~/.claw/sessions/`
  3. 全空 → 区分"有会话但全空"和"无会话"两种错误

- **跨工作区恢复**（`session_control.rs:264-278`）：
  - 别名引用时允许跨 workspace，打印 note 提示来源
  - 显式引用时执行 workspace 校验，不匹配则报错

- **resume 时可执行的命令**：仅 `resume_supported: true` 的命令（`commands/src/lib.rs:1910-1914`），不支持的命令返回 "not yet implemented" + exit code 2

- **session-path 参数必填**：`/resume` 在 REPL 中要求参数（`commands/src/lib.rs:1368-1369`），通过 `require_remainder` 校验

- **持久化格式**：
  - JSONL 格式优先（`save_to_path` 调用 `render_jsonl_snapshot`）（`session.rs:231`）
  - 加载时自动识别格式（JSON 对象 → 旧格式；JSONL → 新格式）（`session.rs:263-276`）
  - 保存时原子写入 + 旋转日志清理

## 依赖关系

- **上游**（本功能依赖）：
  - `runtime::session::Session` — 会话数据结构 + 序列化/反序列化
  - `runtime::session_control::SessionStore` — 会话命名空间管理 + 引用解析
  - `commands::SlashCommand::Resume` — 命令解析
  - `runtime::compact` — 恢复后可能触发自动压缩

- **下游**（依赖本功能）：
  - `/session list` — 列出可恢复的会话
  - `/branch` — fork 的会话可被 resume
  - `--resume` + `/compact` — 组合使用压缩旧会话
  - `/export` — 恢复后可导出

## 当前实现摘要

- **核心文件**：
  - `rust/crates/commands/src/lib.rs:1080-1082` — Resume 枚举定义
  - `rust/crates/commands/src/lib.rs:1368-1370` — `/resume` 解析（require_remainder）
  - `rust/crates/commands/src/lib.rs:117-122` — SlashCommandSpec（summary + argument_hint）
  - `rust/crates/rusty-claude-cli/src/main.rs:3338` — `parse_resume_args` CLI 参数解析
  - `rust/crates/rusty-claude-cli/src/main.rs:5020` — `resume_session` CLI 入口（加载+执行命令）
  - `rust/crates/rusty-claude-cli/src/main.rs:8480` — REPL 内 `resume_session`（切换 runtime）
  - `rust/crates/runtime/src/session_control.rs:20-26` — SessionStore 定义
  - `rust/crates/runtime/src/session_control.rs:257-286` — `load_session_excluding`（核心加载逻辑）
  - `rust/crates/runtime/src/session.rs:231` — `save_to_path`（JSONL 序列化+原子写入）
  - `rust/crates/runtime/src/session.rs:263` — `load_from_path`（JSON/JSONL 自动识别）

- **代码行数**：
  - 命令解析：~50 行（parse_resume_args + SlashCommand 分支）
  - CLI resume 执行：~140 行（resume_session 函数 + 错误分类）
  - REPL resume：~40 行
  - SessionStore：~520 行（session_control.rs 整体）
  - Session 持久化：~50 行（save_to_path + load_from_path）

- **测试覆盖**：
  - `session_control.rs`：21 个单元测试（含 workspace 隔离、latest 解析、跨 workspace）
  - `session.rs`：21 个单元测试（含序列化/反序列化/fork/compaction）
  - `tests/resume_slash_commands.rs`：13 个集成测试（端到端 CLI resume 行为）
