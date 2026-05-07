# Symphony 架构设计文档

> 本文档面向希望深入理解 Symphony Elixir 实现内部结构的开发者。规范层面的权威文档是仓库根目录的 `SPEC.md`。

---

## 1. 系统概述

Symphony 是一个**长期运行的编排服务**，持续从 Linear 看板拉取待办工作，为每个 Issue 创建隔离的工作区，并在工作区内启动 Codex 编码代理执行任务。

```
Linear Board ──poll──▶ Orchestrator ──dispatch──▶ AgentRunner ──launch──▶ Codex (app-server)
                           │                           │
                     retry/reconcile            workspace hooks
                           │
                     StatusDashboard / HttpServer
```

核心设计原则：

- **单一状态权威**：`Orchestrator` 是唯一可变更调度状态的组件，所有工作进程的结果都通过消息回传给它。
- **工作区隔离**：每个 Issue 拥有独立的文件系统目录，Codex 进程的 `cwd` 被强制限定在该目录内。
- **配置即代码**：`WORKFLOW.md` 是运行时配置的唯一来源，包含 YAML 前置元数据和 Liquid 提示词模板，支持热重载。
- **无持久化调度状态**：调度状态仅存于内存；服务重启后通过重新轮询 Linear 和复用已有工作区自动恢复。

---

## 2. OTP 监督树

`SymphonyElixir.Application` 以 `:one_for_one` 策略启动以下子进程（顺序即依赖关系）：

```
SymphonyElixir.Supervisor
├── Phoenix.PubSub              # 发布/订阅总线（dashboard 实时更新）
├── Task.Supervisor             # 名称: SymphonyElixir.TaskSupervisor
│   └── [AgentRunner tasks]     # 每个 Issue 对应一个动态 Task
├── WorkflowStore               # GenServer：缓存并热重载 WORKFLOW.md
├── Orchestrator                # GenServer：调度主循环 + 全部运行状态
├── HttpServer                  # 可选：Phoenix/Bandit Web 服务
└── StatusDashboard             # 终端状态渲染
```

**关键约束**：`WorkflowStore` 必须在 `Orchestrator` 之前启动，因为 `Orchestrator.init/1` 在初始化时就会调用 `Config.settings!()` 读取配置。

---

## 3. 配置与工作流管道

### 3.1 数据流

```
WORKFLOW.md 文件
    │
    ▼
WorkflowStore.current()        # 轮询文件变更（每 1 秒），缓存最后有效版本
    │
    ▼
Workflow.load/1                # 解析 YAML 前置元数据 + Liquid 模板体
    │  {:ok, %{config: map(), prompt_template: string()}}
    ▼
Config.Schema.parse/1          # Ecto 嵌套 Schema 验证、类型转换、默认值填充
    │  {:ok, %Config.Schema{}}
    ▼
Config.settings!()             # 全局可调用的类型化访问器
```

### 3.2 Config.Schema 结构

```
Config.Schema
├── tracker:   kind, endpoint, api_key ($VAR 解析), project_slug, assignee,
│              active_states, terminal_states
├── polling:   interval_ms (默认 30000)
├── workspace: root (支持 ~、$VAR、相对路径)
├── hooks:     after_create, before_run, after_run, before_remove, timeout_ms
├── agent:     max_concurrent_agents, max_turns, max_retry_backoff_ms,
│              max_concurrent_agents_by_state
├── codex:     command, approval_policy, thread_sandbox, turn_sandbox_policy,
│              turn_timeout_ms, read_timeout_ms, stall_timeout_ms
├── worker:    ssh_hosts, max_concurrent_agents_per_host
└── server:    port
```

`$VAR_NAME` 仅在配置值本身包含该形式时才展开，不做全局环境变量覆盖。`tracker.api_key` 默认读取 `LINEAR_API_KEY`。

### 3.3 热重载语义

`WorkflowStore` 每秒比较文件的 `mtime`、`size`、内容哈希三元组。检测到变更后重新解析；解析失败则保留上次有效配置并记录错误日志——**服务不会崩溃**。`Orchestrator` 在每次 tick 开头调用 `refresh_runtime_config/1`，从 `WorkflowStore` 拉取最新的 `poll_interval_ms` 和 `max_concurrent_agents`。

---

## 4. Orchestrator 调度状态机

### 4.1 运行时状态结构

```elixir
%Orchestrator.State{
  poll_interval_ms:      integer,       # 当前生效的轮询间隔
  max_concurrent_agents: integer,       # 当前生效的全局并发上限
  running:     %{issue_id => running_entry},   # 正在运行的任务
  claimed:     MapSet<issue_id>,               # 已被预占（运行中或等待重试）
  retry_attempts: %{issue_id => retry_entry},  # 重试队列
  completed:   MapSet<issue_id>,               # 已完成（仅记账用）
  codex_totals: %{input_tokens, output_tokens, total_tokens, seconds_running},
  codex_rate_limits: map | nil
}
```

`running_entry` 关键字段：`pid`、`ref`（monitor ref）、`identifier`、`issue`（最新 Issue 快照）、`session_id`、`turn_count`、`started_at`、`last_codex_timestamp`、`worker_host`、`workspace_path`、Token 计数。

### 4.2 Tick 序列

```
on_tick:
  1. refresh_runtime_config          ← 从 WorkflowStore 拉取最新配置
  2. reconcile_stalled_running_issues ← 超时检测
  3. fetch_issue_states_by_ids       ← 刷新所有运行中 Issue 的状态
     ├─ terminal → terminate + cleanup workspace
     ├─ active   → 更新运行条目中的 issue 快照
     └─ other    → terminate（不清理工作区）
  4. Config.validate!                ← 调度前置校验
  5. Tracker.fetch_candidate_issues  ← 获取候选 Issue
  6. sort_issues_for_dispatch        ← 优先级↑ → 创建时间↑ → identifier 字典序
  7. choose_issues（循环 dispatch）  ← 检查 claimed/running/slots
  8. notify_dashboard
  9. schedule_tick(poll_interval_ms)
```

### 4.3 调度资格检查（`should_dispatch_issue?`）

一个 Issue 满足以下**全部**条件才会被派发：

1. 具有非空 `id`、`identifier`、`title`、`state`
2. `state` 属于 `active_states` 且不属于 `terminal_states`
3. 不在 `claimed` 或 `running` 中
4. 全局并发槽 `available_slots > 0`
5. 该状态的 per-state 并发槽有空余
6. SSH worker 有可用容量（若配置了 `worker.ssh_hosts`）
7. 状态为 `Todo` 时：所有 blocker 均为终止状态

在真正 spawn Task 前还会重新从 Linear 拉取该 Issue 的最新状态做二次校验（`revalidate_issue_for_dispatch`），避免派发到已离开活跃状态的 Issue。

### 4.4 重试与退避

| 类型 | 延迟公式 |
|------|---------|
| 正常退出后的续接重试 | 固定 1 000 ms（attempt=1） |
| 异常退出后的指数退避 | `min(10_000 × 2^(attempt-1), max_retry_backoff_ms)` |

每个 `retry_entry` 携带 `retry_token`（`make_ref()`），用于防止过期定时器触发重复重试。

---

## 5. 工作区管理

### 5.1 路径计算

```
workspace_root = Config.settings!().workspace.root   (绝对路径)
safe_id        = issue.identifier 中非 [A-Za-z0-9._-] 的字符替换为 _
workspace_path = Path.join(workspace_root, safe_id)
```

`PathSafety` 模块在创建和 Codex 启动前均会验证 `workspace_path` 是否以 `workspace_root` 为前缀，防止路径穿越。

### 5.2 钩子执行顺序

```
create_for_issue
  ├─ ensure_workspace (mkdir 或复用)
  └─ after_create hook（仅新建时执行；失败 → abort workspace creation）

AgentRunner.run
  ├─ before_run hook（每次尝试；失败 → abort attempt）
  ├─ [Codex 会话循环]
  └─ after_run hook（always；失败仅记录，不影响结果）

remove_issue_workspaces
  └─ before_remove hook（失败仅记录）
```

钩子以 `bash -lc <script>` 在工作区目录执行，受 `hooks.timeout_ms` 限制。

### 5.3 SSH 远程工作区

当配置了 `worker.ssh_hosts` 时，工作区操作（创建、验证、钩子执行）通过 `SSH` 模块在远程主机上执行。Codex 进程也通过 SSH stdio 启动，Orchestrator 仍是唯一状态持有者。

---

## 6. Codex AppServer 客户端

### 6.1 会话生命周期

```
AppServer.start_session(workspace)
  ├─ start_port: bash -lc <codex.command>  (Erlang Port，stdio 通信)
  ├─ do_start_session: 发送 initialize + thread/start JSON-RPC 请求
  └─ 返回 session（含 port、thread_id、sandbox 策略等）

AppServer.run_turn(session, prompt, issue, on_message: cb)
  ├─ 发送 turn/start（携带 prompt、workspace cwd、sandbox 策略）
  ├─ 流式读取响应行（每行一个 JSON-RPC 通知）
  │   ├─ 自动审批命令/文件变更（当 approval_policy == "never"）
  │   ├─ linear_graphql 动态工具调用处理
  │   ├─ 提取 token 用量并通过 on_message 回传 Orchestrator
  │   └─ user-input-required → 立即失败（非交互会话）
  └─ 返回 {:ok, map()} | {:error, reason}

AppServer.stop_session(session)
  └─ 关闭 Port
```

### 6.2 Token 计账

优先使用 `thread/tokenUsage/updated.tokenUsage.total`（绝对累计值）。收到新值时计算差值（`delta = new_total - last_reported_total`）并累加到 `Orchestrator.codex_totals`，避免重复计数。详见 `elixir/docs/token_accounting.md`。

### 6.3 动态工具：`linear_graphql`

Codex 会话可调用 `linear_graphql` 工具直接向 Linear GraphQL API 发起请求，使用 Symphony 配置的认证信息，无需将 token 暴露给 Codex 进程。工具输入：

```json
{ "query": "...", "variables": { "optional": "..." } }
```

---

## 7. 线性 API 集成

### 7.1 适配器层次

```
Tracker（协议层）
  └─ adapter() ─┬─ Linear.Adapter（生产）
                └─ Tracker.Memory（测试 double，tracker.kind = "memory"）
```

`Tracker.Memory` 是进程内测试替身，允许单元测试无需真实 Linear 访问即可验证编排逻辑。

### 7.2 关键 GraphQL 操作

| 操作 | 用途 |
|------|------|
| `fetch_candidate_issues` | 按 `active_states` 和 `project_slug` 查询候选 Issue（分页，每页 50 条） |
| `fetch_issues_by_states` | 启动时清理终止态工作区 |
| `fetch_issue_states_by_ids` | reconciliation 刷新运行中 Issue 状态（使用 `[ID!]` 类型） |

Issue 规范化：labels 小写、blockers 提取自 `blocks` 类型的反向关联、priority 强制整数化。

---

## 8. Web 可观测性层（可选）

当 `server.port` 或 `--port` 有效时，`HttpServer` 启动：

```
Phoenix Router
├── /                         LiveView Dashboard（实时推送 via PubSub）
└── /api/v1/
    ├── GET  state             编排器全量快照
    ├── GET  <issue_identifier> Issue 级别调试详情
    └── POST refresh           触发立即轮询（幂等，可合并）
```

`ObservabilityApiController` 和 `DashboardLive` 均只读取 `Orchestrator.snapshot/0`，不持有独立状态，不影响编排正确性。

`ObservabilityPubSub` 封装 `Phoenix.PubSub`，在 Orchestrator 调用 `StatusDashboard.notify_update()` 后向 LiveView 推送更新。

---

## 9. 关键设计决策

### 9.1 为什么用 Elixir/OTP？

- BEAM 的 `Process.monitor` 天然支持"工作进程意外退出 → 自动重试"语义，无需手写守护线程。
- `GenServer` + 消息传递使调度状态始终在单一进程中串行修改，避免竞态条件。
- 热代码加载（开发阶段）不中断正在运行的 Codex 子进程。

### 9.2 无数据库设计

调度状态仅存于 `Orchestrator` 进程内存。重启后：
1. 启动时清理终止态 Issue 的工作区
2. 轮询 Linear 获取当前活跃 Issue
3. 重新派发

代价是重试队列和运行中状态不跨重启保留。工作区文件本身是持久化的，续接运行可复用。

### 9.3 Token 计账去重

Codex 同时发出 `thread/tokenUsage/updated`（绝对累计）和 `turn/completed`（完成时 usage），对同一消耗会重复报告。Symphony 通过 `last_reported_*` 高水位线记录，仅计入超过上次报告值的增量，避免双重计数。

---

## 10. 数据流图（完整路径）

```
Linear API
  │  (GraphQL, 30s 间隔)
  ▼
Orchestrator ──spawn Task──▶ AgentRunner
  ▲                              │
  │  {:codex_worker_update,      ├─ Workspace.create_for_issue
  │   issue_id, update}          ├─ hooks.before_run
  │                              │
  │                         AppServer.start_session
  │                              │  (bash -lc "codex app-server")
  │                              │  JSON-RPC 2.0 over stdio (Erlang Port)
  │                              │
  │                         AppServer.run_turn (循环至 max_turns)
  │                              │
  │◀─────── on_message ──────────┤  token updates, events
  │                              │
  │                         Tracker.fetch_issue_states_by_ids
  │                              │  (每 turn 完成后检查 Issue 状态)
  │                              │
  │                         AppServer.stop_session
  │                              │
  │                         hooks.after_run
  │
  └── Task 正常退出 → schedule continuation retry (1s)
  └── Task 异常退出 → schedule backoff retry

Orchestrator.snapshot()
  │
  ├─▶ StatusDashboard (终端渲染)
  └─▶ HttpServer / LiveView (Web 仪表盘)
```
