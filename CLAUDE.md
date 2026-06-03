# CLAUDE.md

## 项目概述

Nezha 是一款面向 AI 编程智能体（Claude Code、Codex）的桌面任务管理器，提供多项目工作区、实时终端输出、会话自动发现、权限感知执行、Git 集成和用量分析等核心功能。

**技术栈：** React 19 + TypeScript + Vite · Tauri 2 + Rust · xterm.js · Shiki

---

## 开发命令

```bash
pnpm dev            # 启动 Vite 开发服务器（端口 1420）
pnpm build          # tsc 类型检查 + Vite 打包
pnpm lint           # 运行 ESLint
pnpm test           # 运行 Vitest
pnpm tauri dev      # 启动完整桌面应用（自动启动开发服务器）
pnpm tauri build    # 构建生产环境桌面二进制包
```

Rust 后端位于 `src-tauri/`，修改后需重启 `tauri dev`。

---

## 架构概览

### 前端（`src/`）

| 文件 | 职责 |
|------|------|
| `App.tsx` | 根组件；持有所有状态（projects、tasks、buffers）及 Tauri 事件监听器 |
| `types.ts` | TypeScript 接口的权威定义——修改数据结构时优先编辑此文件 |
| `styles/index.ts` + `styles/*` | 模块化 CSS-in-JS 样式入口；按 layout / panels / task / terminal / dialogs / common 拆分 |
| `App.css` | 仅用于暗色/亮色主题的 CSS 自定义属性 |
| `utils.ts` | UI 工具函数：头像颜色生成、路径缩短、localStorage 辅助方法 |

### 后端（`src-tauri/src/`）

`lib.rs` 是入口点，注册所有模块和 Tauri 命令处理列表：

| 模块 | 职责 |
|------|------|
| `pty.rs` | PTY 创建/读写（`run_task`、`resume_task`、`cancel_task`、`send_input` 等） |
| `session.rs` | 会话文件监听、终端输出提取、`read_session_messages` |
| `storage.rs` | 文件持久化（`load/save_projects`、`load/save_project_tasks`） |
| `fs.rs` | 文件系统命令（`read_dir_entries`、`read_file_content` 等） |
| `git.rs` | Git 集成：状态、分支、日志、差异、暂存/提交/推送 |
| `analytics.rs` | 会话 JSONL 指标解析 |
| `config.rs` | 项目级 `.nezha/config.toml` 管理 |
| `app_settings.rs` | 应用级智能体路径与版本管理 |

详细架构参见 [[AGENTS.md]]。

---

## 编码规范

### TypeScript
- 已开启严格模式（`tsconfig.json`）。禁止使用 `any`，应扩展 `types.ts` 代替。
- Tauri 命令使用 `invoke<ReturnType>()` 类型化。

### Rust
- Tauri 命令按模块文件组织。新增命令须在 `lib.rs` 的 `invoke_handler!` 中注册。
- 优先使用 `parking_lot::Mutex`，锁作用域尽可能短。
- 优先使用 `tauri::Emitter` 向前端推送事件，而非从命令中返回大体积数据。
- 所有重型/阻塞操作必须使用 `tokio::task::spawn_blocking` 或 `tokio::spawn`。

### 样式
- 所有样式放在 `src/styles/` 目录，通过 `src/styles/index.ts` 聚合导出。
- 不要创建独立的业务 `.css` 文件，不要使用行内 `style={{}}`（真正一次性的除外）。
- 主题变量在 `App.css` 中定义为 CSS 自定义属性，在 `styles/*` 中通过 `var(--name)` 引用。

### 状态管理
- 不引入外部状态库（Redux、Zustand 等）。核心状态存活在 `App.tsx`，通过 props 传递。
- 异步更新通过 Tauri 事件（`listen()`）驱动。

---

## 关键约束

- **不要引入 CSS 框架**（Tailwind 等）。
- **不要在未经讨论的情况下引入状态管理库**。
- **交互式 UI 原语优先使用 Radix UI**（`@radix-ui/react-select` 等），不要用原生 `<select>`、`<dialog>`。图标用 `lucide-react`。
- **`read_file_content` 不要读取超过 2 MB 的文件**（Rust 侧强制）。
- **不要在未考虑迁移方案的情况下修改文件存储 schema**（`~/.nezha/`）。
- **不要阻塞 Tauri 主线程**——所有重型操作必须通过 `tokio::task::spawn_blocking`。
- **修改 Task 数据结构时，必须同步更新 `types.ts`（TypeScript）和 `storage.rs` 中的 `Task` 结构体（Rust）**。

---

## 性能防劣化规则

> 新增代码必须遵守以下规则，存量代码逐步修复。

### 前端
- 组件必须控制渲染范围，列表行组件使用 memo 化。
- 高频事件回调中避免 `setState`——沿用 Channel + RAF 批量写入策略。
- `persistProjectTasks` 必须防抖（300-500ms）。
- 长列表必须虚拟化（5000+ 消息、1000+ 文件变更场景）。
- `marked()` 禁止同步调用大文本（>10KB 应异步渲染或 memoize）。
- @提及搜索必须防抖（200ms）或使用 `startTransition`。
- CodeMirror / Shiki 语言包应按需加载（动态 `import()`）。

### 后端
- Tauri async 命令内禁止直接调用阻塞操作——必须包裹 `tokio::task::spawn_blocking`。
- PTY 读取缓冲区应至少 32KB-64KB。
- 持锁期间禁止执行 I/O——先 clone/取出资源再释放锁。
- `read_session_messages` 禁止全文件一次性加载——应改为流式逐行读取。
- `list_project_files` 应合并 git 命令（`git ls-files -c -o --exclude-standard`）。

### 安全
- 所有接受路径参数的 Tauri 命令必须验证路径合法性（位于项目目录内）。
- Mutex 获取禁止裸 `.unwrap()`。

### 组件规模
- 单个组件文件不应超过 400 行。

---

## 数据模型（Task）

```typescript
// 状态生命周期：todo → pending → running ↔ input_required → done | failed | cancelled

interface Task {
  id: string;
  projectId: string;
  name?: string;
  prompt: string;
  agent: "claude" | "codex";
  permissionMode: "ask" | "auto_edit" | "full_access";
  status: TaskStatus;
  createdAt: number;
  attentionRequestedAt?: number;
  starred?: boolean;
  failureReason?: string;
  claudeSessionId?: string;
  claudeSessionPath?: string;
  codexSessionId?: string;
  codexSessionPath?: string;
}
```

**持久化存储：**
- `~/.nezha/projects.json` — Project[]
- `~/.nezha/projects/<projectId>/tasks.json` — Task[]
- 主题及 UI 偏好存储于 localStorage

---

## 项目配置

每个项目首次打开时自动创建 `.nezha/config.toml`：

```toml
[agent]
default = "claude"        # 新任务的默认智能体
prompt_prefix = ""        # 拼接到每个任务提示词前面的文本
claude_version = ""
codex_version = ""

[git]
commit_prompt = "..."     # generate_commit_message 使用的提示词
```

---

## 会话自动发现

- **Claude Code**：`~/.claude/projects/<encoded-path>/*.jsonl`
- **Codex**：`<project-path>/.codex/sessions/*.jsonl`

会话通过项目路径、提示词文本和创建时间戳与任务匹配。兜底方案是从智能体终端输出中提取会话 ID。
