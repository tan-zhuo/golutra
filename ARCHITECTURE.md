# golutra 架构设计文档

## 概述

golutra 是一个多智能体桌面工作空间，允许用户在统一的可视界面中并行管理多个 AI CLI（Claude Code、Gemini CLI、Codex、OpenCode、Qwen 等）。技术栈为 **Vue 3（前端）+ Rust（后端）**，通过 **Tauri 2** 封装为跨平台桌面应用（Windows / macOS / Linux）。

---

## 整体分层

```
┌─────────────────────────────────────────────────────┐
│                    前端 (Vue 3)                      │
│  app/  features/  stores/  shared/                  │
└─────────────────────┬───────────────────────────────┘
                      │  Tauri IPC（invoke / event）
┌─────────────────────▼───────────────────────────────┐
│                  后端 (Rust)                         │
│                                                     │
│  ui_gateway  ←→  application                        │
│       ↕                ↕                            │
│  orchestration  ←→  terminal_engine                 │
│       ↕                ↕                            │
│  message_service  ←→  runtime                       │
│       ↕                                             │
│  contracts / ports / platform                       │
└─────────────────────────────────────────────────────┘
```

---

## 后端模块详解（Rust / src-tauri）

### 1. `runtime/` — 基础设施层

负责操作系统级能力，不含业务逻辑。

| 子模块 | 职责 |
|--------|------|
| `pty.rs` | 跨平台 PTY 启动（`portable-pty`），Windows 路径兼容处理 |
| `storage.rs` | JSON 与二进制数据的本地持久化（app_data_dir / app_cache_dir） |
| `settings.rs` | 用户设置读写服务 |
| `command_center.rs` | 应用级命令总线 |
| `command_ipc.rs` | 进程间命令通信服务（`interprocess`），供 CLI shim 连接 |
| `state.rs` | 全局 AppState（多窗口管理、活跃窗口追踪） |

**关键依赖**：`portable-pty`、`redb`（嵌入式 KV 数据库）、`interprocess`

---

### 2. `terminal_engine/` — 终端引擎层

核心子系统，负责 PTY IO 管线、终端仿真、语义分析与会话生命周期。

```
terminal_engine/
├── session/          # 会话管理（PTY 线程、状态机、快照、派发队列）
│   ├── launch.rs     # 会话启动与 shim 注入
│   ├── poller.rs     # 后台状态轮询（500ms 间隔）
│   ├── polling/      # 规则驱动的轮询动作（post_ready、status_fallback、semantic_flush）
│   ├── semantic_worker.rs  # 异步语义处理线程
│   ├── snapshot_service.rs # 快照生成与 attach 服务
│   ├── trigger/      # 事件触发器（FactEvent → 语义派发）
│   ├── keyboard_input.rs   # 键盘输入注入
│   └── state.rs      # 会话注册表与状态枚举
├── emulator.rs       # 终端仿真器封装（wezterm-term / tattoy-wezterm-term）
├── semantic.rs       # 语义层：将终端输出整理为 TerminalMessagePayload
├── filters/          # 输出过滤器（按 CLI 类型加载规则，过滤 ANSI/噪音）
│   ├── profiles/     # 各 CLI 的过滤规则配置（generic 等）
│   ├── rules/        # 具体过滤规则（prompt_block 等）
│   └── registry.rs   # 规则注册与运行时选择
├── default_members/  # 内置 CLI 成员配置注册
│   ├── claude.rs / gemini.rs / codex.rs / opencode.rs / qwen.rs / shell.rs
│   └── registry.rs   # 默认命令解析与成员配置
└── models.rs         # IPC 数据结构（Output/Exit/Status/Snapshot 载荷）
```

#### 会话状态机

```
Offline → Launching → Online ⟺ Working → Offline
                                    ↓
                              (语义分析 → chat flush)
```

- **Launching**：PTY 启动后等待 shim `OSC 633;A` 信号
- **Online**：就绪，等待输入
- **Working**：检测到活跃输出，触发语义收集
- 状态回落有防抖（`STATUS_IDLE_DEBOUNCE_MS = 1000ms`）和静默门禁（`STATUS_WORKING_SILENCE_TIMEOUT_MS = 4500ms`）

#### 重要常量

| 常量 | 值 | 说明 |
|------|----|------|
| `SESSION_SCROLLBACK_LINES` | 2000 | PTY 快照保留行数 |
| `SEMANTIC_SCROLLBACK_LINES` | 5000 | 语义层历史行数 |
| `DISPATCH_QUEUE_LIMIT` | 32 | 派发队列上限 |
| `CHAT_PENDING_FORCE_FLUSH_MS` | 30000ms | chat flush 超时兜底 |
| `STATUS_POLL_INTERVAL_MS` | 500ms | 状态轮询频率 |

---

### 3. `message_service/` — 消息服务层

负责聊天记录持久化与消息投递流水线。

```
message_service/
├── chat_db/          # 聊天数据库（redb KV + bincode 序列化）
│   ├── store.rs      # ChatDbManager（按工作区隔离的 DB 实例池）
│   ├── types.rs      # 消息结构体定义
│   ├── read.rs / write.rs  # CRUD 操作
│   ├── outbox.rs     # 发件箱（待发送消息队列）
│   └── terminal_session_map.rs  # 终端会话 ↔ 聊天会话映射
├── pipeline/         # 消息投递流水线
│   ├── normalize.rs  # 规范化（清洗载荷）
│   ├── dispatch.rs   # 投递计划（确定目标）
│   ├── policy.rs     # 策略评估（过滤规则）
│   ├── throttle.rs   # 流量控制
│   └── reliability.rs # 可靠投递（stream / final 两路）
├── project_data.rs   # 项目元数据读写
└── project_members.rs # 项目成员数据访问
```

#### 消息流水线

```
TerminalMessagePayload
    → normalize（规范化）
    → plan（投递计划）
    → policy（策略评估）
    → throttle（限流）
    → deliver_stream / deliver_final（可靠投递）
        → TerminalMessageTransport（Tauri event）
        → TerminalMessageRepository（chat_db 写入）
```

---

### 4. `orchestration/` — 编排层

跨模块自动化：将聊天消息翻译为终端派发指令。

| 模块 | 职责 |
|------|------|
| `dispatch.rs` | 核心编排逻辑：mention 解析 → 目标成员 → 确保会话存在 → 派发 |
| `chat_dispatch_batcher.rs` | 批量派发器，防止同一会话重复堆积 |
| `chat_outbox.rs` | 后台 outbox worker，定期处理待发消息 |
| `terminal_friend_invite.rs` | 终端邀请成员的编排流程 |

#### 聊天派发流程

```
用户在 Chat 发送消息
    → orchestrate_chat_dispatch()
    → 读取会话成员列表（chat_db）
    → 读取项目成员配置（project_data）
    → 解析派发目标（DM → 对方；Channel → @mentions）
    → 过滤无终端配置的成员
    → 对每个目标：ensure_backend_session() + batcher.enqueue()
    → ChatDispatchBatcher → terminal_dispatch_chat()
    → PTY write
```

---

### 5. `application/` — 应用层

跨入口（UI / CLI）复用的业务用例，避免 Tauri 命令与 CLI 分叉。

| 模块 | 职责 |
|------|------|
| `chat.rs` | 聊天相关业务逻辑 |
| `command.rs` | 命令处理业务逻辑 |
| `project.rs` | 工作区/项目管理逻辑 |

---

### 6. `ui_gateway/` — UI 接口层

向前端暴露 Tauri 命令，是后端对外的唯一边界。

| 模块 | 职责 |
|------|------|
| `commands.rs` | 汇总并导出所有 `#[tauri::command]` |
| `terminal.rs` | 终端相关命令（create/write/close/attach 等） |
| `message_pipeline.rs` | `UiTerminalMessagePipeline`（Transport + Repository 实现） |
| `terminal_events.rs` | `UiTerminalEventPort`（向前端 emit Tauri 事件） |
| `terminal_session_repository.rs` | 会话仓库的 UI 实现 |
| `app.rs` | 窗口管理、托盘、系统级 UI 操作 |
| `notification.rs` | 系统通知与角标管理 |
| `monitoring.rs` | 诊断日志向前端暴露 |
| `skills.rs / project_skills.rs` | Skill/技能注册与项目级 Skill 管理 |

---

### 7. `contracts/` — 契约层

跨层共享的数据结构（无业务逻辑），确保编译期类型安全。

| 模块 | 内容 |
|------|------|
| `terminal_message.rs` | `TerminalMessagePayload`（终端 → 聊天的消息体） |
| `chat_dispatch.rs` | `ChatDispatchPayload`（聊天 → 终端的派发体） |

---

### 8. `ports/` — 端口层

接口定义（Rust trait），隔离实现细节，便于依赖倒置与测试替换。

| 端口 | 用途 |
|------|------|
| `terminal_event` | 终端事件发布 |
| `terminal_session` | 会话状态查询 |
| `terminal_message` | 消息写入/读取 |
| `terminal_dispatch_gate` | 派发门禁（限流/批处理） |
| `message_service` | 消息传输与仓库抽象 |
| `settings` | 配置读取抽象 |

---

### 9. `platform/` — 平台层

平台差异与运维能力。

| 模块 | 职责 |
|------|------|
| `paths.rs` | 平台路径解析（log dir 等） |
| `activation.rs` | 许可证激活状态 |
| `updater.rs` | 自动更新 |
| `monitoring/` | 诊断日志（后端 gate + diagnostics） |

---

## 前端模块详解（Vue 3 / src）

### 1. `app/` — 应用根

| 文件 | 职责 |
|------|------|
| `App.vue` | 根组件，路由切换、模态层、Toast 层 |
| `useWorkspaceBootstrap.ts` | 工作区启动流程（加载项目数据、初始化成员会话） |
| `useAppKeybinds.ts` | 全局快捷键注册 |

### 2. `features/` — 功能模块

#### `chat/`
- `chatStore.ts`：Pinia store，消息列表、发送逻辑、会话管理
- `chatBridge.ts`：封装 Tauri IPC，订阅聊天事件
- `ChatInterface.vue`：主聊天界面
- `components/`：ChatInput、MessagesList、ChatSidebar、MembersSidebar 等
- `modals/`：InviteFriends、SkillManagement、ManageMember 等弹窗

#### `terminal/`
- `terminalBridge.ts`：**终端 IPC 适配层**，封装所有 `invoke` 与 `listen`，维护输出缓冲（2000条）与 ACK 批处理（5000字节 / 50ms）
- `terminalStore.ts`：终端标签页状态
- `terminalMemberStore.ts`：成员 ↔ 会话映射
- `TerminalWorkspace.vue`：终端工作区布局
- `TerminalPane.vue`：单个终端面板（xterm.js 集成）

#### `workspace/`
- `projectStore.ts`：项目成员、配置数据
- `workspaceStore.ts`：工作区切换与持久化

#### `skills/`
- `skillLibrary.ts`：技能定义库
- `skillsBridge.ts`：技能 IPC 适配

### 3. `stores/` — 全局 Pinia Stores

| Store | 职责 |
|-------|------|
| `terminalOrchestratorStore.ts` | **前端编排层**：消息 mention 解析 → 目标成员 → 派发队列 |
| `notificationOrchestratorStore.ts` | 通知路由与去重 |
| `navigationStore.ts` | 导航状态（侧边栏选中项） |
| `toastStore.ts` | Toast 消息队列 |
| `terminalSnapshotAuditStore.ts` | 快照审计日志 |

### 4. `shared/` — 跨功能共享

| 目录 | 内容 |
|------|------|
| `tauri/` | Tauri IPC 适配（terminal、storage、notifications、avatars、windows、projectData） |
| `monitoring/` | 前端诊断日志（passiveMonitor、frontendGate、logger） |
| `keyboard/` | 全局键盘快捷键系统（registry + controller + profiles） |
| `context-menu/` | 右键菜单系统（registry + controller） |
| `types/` | 共享 TypeScript 类型（terminal、conversation、memberDisplay、terminalDispatch） |
| `constants/` | terminalCatalog（CLI 类型枚举）、terminalCallChains、avatars、timeZones |
| `utils/` | terminal、avatar、memberDisplay 工具函数 |
| `components/` | SidebarNav、AvatarBadge、ToastStack |

---

## IPC 通信协议

### 后端 → 前端（Tauri Events）

| 事件名 | 载荷类型 | 触发时机 |
|--------|---------|---------|
| `terminal-output` | `{ terminalId, data, seq }` | PTY 有输出，seq 单调递增 |
| `terminal-exit` | `{ terminalId, code, signal }` | 进程退出 |
| `terminal-status-change` | `{ terminalId, status, memberId, workspaceId }` | 会话状态变更（Online/Working/Offline） |
| `terminal-chat-output` | `TerminalChatPayload` | 语义消息最终版（final/snapshot） |
| `terminal-message-stream` | `TerminalChatPayload` | 语义消息流式版（stream） |
| `terminal-error` | `{ terminalId, error, fatal }` | 会话错误，fatal=true 时不可恢复 |

### 前端 → 后端（Tauri Commands）

| 命令 | 参数 | 说明 |
|------|------|------|
| `terminal_create` | cols/rows/cwd/memberId/terminalType 等 | 创建会话，返回 terminalId |
| `terminal_write` | terminalId, data | 向 PTY 写入用户输入 |
| `terminal_ack` | terminalId, count | 确认已消费字节数（流控） |
| `terminal_resize` | terminalId, cols, rows | 调整终端尺寸 |
| `terminal_dispatch` | terminalId, data, context | 聊天派发到终端 |
| `terminal_attach` | terminalId | 获取快照（UI attach 时恢复内容） |
| `terminal_close` | terminalId, preserve | 关闭会话 |
| `terminal_set_active` | terminalId, active | 标记 UI 激活状态 |
| `terminal_snapshot_lines` | terminalId | 获取文本行快照（一致性校验） |

---

## 数据流全景

```
用户在 Chat 输入消息
    ↓
chatStore.sendMessage()
    ↓  本地写入 chatStorage
terminalOrchestratorStore.dispatchConversationToTerminals()
    ↓  解析 mention 目标
terminalMemberStore.enqueueTerminalDispatch()
    ↓  invoke('terminal_dispatch')
─────────── Tauri IPC ───────────
ui_gateway/commands.rs
    ↓
orchestration/dispatch.rs → ensure_backend_session() → terminal_dispatch_chat()
    ↓
terminal_engine/session: 写入派发队列 → PTY write
    ↓
PTY 子进程（AI CLI）运行，产生输出
    ↓
PTY reader 线程读取 → TerminalOutputPayload → emit('terminal-output')
    ↓  同时送入语义 worker
semantic_worker → SemanticState → build_semantic_payload()
    ↓  经 message_service/pipeline 流水线
chat_db 写入 + emit('terminal-chat-output' / 'terminal-message-stream')
─────────── Tauri IPC ───────────
terminalBridge 监听事件
    ↓
TerminalPane.vue 渲染输出 / chatStore 更新消息列表
```

---

## 关键设计决策

### 1. Ports & Adapters（六边形架构）
`ports/` 层定义 trait 接口，`ui_gateway/` 提供基于 Tauri 的实现。终端引擎只依赖 Port trait，不直接引用 Tauri，保持可测试性。

### 2. 双路消息输出
- **PTY 原始流**：`terminal-output` 事件，直接驱动 xterm.js 渲染，低延迟
- **语义消息流**：`terminal-chat-output` / `terminal-message-stream`，经语义层整理后写入聊天记录，带结构化 meta

### 3. 流控与背压
- 前端 ACK 机制：每 5000 字节或 50ms 批量确认，防止 PTY 输出洪峰
- 后端 `DISPATCH_QUEUE_LIMIT = 32`：防止 Working 状态无限堆积
- `ChatDispatchBatcher`：防止同一会话短时间内重复派发

### 4. 语义状态机（polling rules）
会话状态轮询由规则链驱动：
- `post_ready`：启动后置流程（发送邀请命令等）
- `semantic_flush`：Working 静默后将语义缓冲 flush 为聊天消息
- `status_fallback`：Working 超时回落到 Online

### 5. 多 CLI 适配（default_members + filters）
每种 CLI（Claude/Gemini/Codex 等）有独立的：
- 默认启动命令（`default_members/registry.rs`）
- 输出过滤规则（`filters/profiles/`）：屏蔽特定 ANSI 序列、提示符块等噪音

### 6. 工作区隔离
- 每个工作区有独立的 `chat_db`（redb 文件），按 `workspace_id` 隔离
- 会话也按 `(member_id, workspace_id)` 查找，避免跨工作区串扰

---

## 技术栈汇总

| 层次 | 技术 |
|------|------|
| 前端框架 | Vue 3 + Composition API |
| 前端状态 | Pinia |
| 前端构建 | Vite |
| 前端样式 | Tailwind CSS |
| 终端渲染 | xterm.js（6.1.0-beta） |
| 桌面框架 | Tauri 2 |
| 后端语言 | Rust（edition 2021） |
| PTY | portable-pty 0.9 |
| 终端仿真 | tattoy-wezterm-term（wezterm-term fork） |
| 数据库 | redb 2（嵌入式 KV） |
| 序列化 | serde_json + bincode |
| ID 生成 | ULID |
| IPC | interprocess（Unix socket / Named Pipe） |
| 国际化 | vue-i18n（中文 / 英文） |
