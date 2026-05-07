# Symphony 用户快速入门指南

Symphony 会监听你的 Linear 看板，自动为每张工单启动 Codex 编码代理，直到工作完成。本指南帮助你在 30 分钟内完成首次运行。

---

## 前提条件

| 依赖 | 说明 |
|------|------|
| [mise](https://mise.jdx.dev/) | 用于管理 Elixir/Erlang 版本 |
| Elixir 1.19 + OTP 28 | 由 mise 自动安装 |
| [Codex CLI](https://github.com/openai/codex) | 需要支持 `codex app-server` 子命令 |
| Linear 账号 | 需要一个项目和个人 API Token |

验证 Codex 已正确安装：

```bash
codex app-server --help
```

---

## 第一步：安装 Symphony

```bash
git clone https://github.com/openai/symphony
cd symphony/elixir

mise trust && mise install      # 安装 Elixir/Erlang
mix setup                       # 拉取依赖
mix build                       # 编译为可执行文件 bin/symphony
```

---

## 第二步：获取 Linear API Token

1. 打开 Linear → 右上角头像 → **Settings → Security & access → Personal API keys**
2. 创建新 Token，复制保存
3. 设置环境变量：

```bash
export LINEAR_API_KEY="lin_api_xxxxxxxxxxxxxxxx"
```

建议将此行加入 `~/.bashrc` 或 `~/.zshrc` 以持久生效。

---

## 第三步：获取 Linear 项目 Slug

1. 在 Linear 中右键点击目标项目，选择 **Copy link**
2. URL 格式类似：`https://linear.app/your-org/project/my-project-abc123def456`
3. Slug 是 URL 最后一段：`my-project-abc123def456`

---

## 第四步：配置 WORKFLOW.md

将 `elixir/WORKFLOW.md` 复制到你的目标代码仓库，或直接编辑本仓库中的副本。

最简配置示例：

```yaml
---
tracker:
  kind: linear
  project_slug: "my-project-abc123def456"   # ← 替换为你的 Slug
workspace:
  root: ~/code/my-workspaces                # ← 工作区存储路径
hooks:
  after_create: |
    git clone --depth 1 git@github.com:your-org/your-repo.git .
agent:
  max_concurrent_agents: 3    # 同时运行的最大代理数
  max_turns: 20               # 单次会话最大对话轮次
codex:
  command: codex app-server
  approval_policy: never      # 高信任模式：自动审批所有操作
---

你正在处理 Linear 工单 `{{ issue.identifier }}`。

标题：{{ issue.title }}

描述：
{{ issue.description }}
```

**关键字段说明：**

| 字段 | 说明 |
|------|------|
| `tracker.project_slug` | Linear 项目 Slug（必填） |
| `workspace.root` | 每个 Issue 的工作区存储根目录（支持 `~` 和 `$VAR`） |
| `hooks.after_create` | 工作区首次创建时执行的脚本，通常用于 `git clone` |
| `hooks.before_run` | 每次代理尝试前执行，如安装依赖 |
| `agent.max_concurrent_agents` | 并发代理上限，建议从 2–3 开始 |
| `codex.approval_policy` | `never` = 全自动；`on-request` = 需要操作员确认 |

---

## 第五步：启动 Symphony

```bash
cd symphony/elixir
./bin/symphony /path/to/your/WORKFLOW.md
```

可选参数：

```bash
# 同时启动 Web 仪表盘（访问 http://localhost:4000）
./bin/symphony ./WORKFLOW.md --port 4000

# 指定日志存储目录
./bin/symphony ./WORKFLOW.md --logs-root /var/log/symphony
```

启动成功后，终端会显示状态面板，类似：

```
Symphony  ·  checking now…
─────────────────────────────────────
  running   0    retrying  0
  tokens    0    runtime   0s
─────────────────────────────────────
```

---

## 第六步：触发第一个任务

1. 在 Linear 中将一张工单的状态改为 `Todo` 或 `In Progress`（即你在 `active_states` 中配置的状态）
2. Symphony 在下一次轮询时（默认 30 秒，可通过 `polling.interval_ms` 调整）将自动检测并派发该工单
3. 状态面板中会出现正在运行的代理条目

---

## Web 仪表盘（可选）

使用 `--port` 启动后，访问：

| 地址 | 说明 |
|------|------|
| `http://localhost:4000/` | 实时仪表盘（LiveView） |
| `http://localhost:4000/api/v1/state` | JSON 格式系统快照 |
| `http://localhost:4000/api/v1/MT-123` | 指定 Issue 的调试详情 |
| `POST /api/v1/refresh` | 触发立即轮询 |

---

## 常用配置场景

### 场景一：仅处理指定标签的工单

在 WORKFLOW.md 提示词中加入过滤说明，或使用 Linear 过滤视图仅暴露特定工单。

### 场景二：使用 SSH 远程 Worker

```yaml
worker:
  ssh_hosts:
    - "user@build-server-01.example.com"
    - "user@build-server-02.example.com"
  max_concurrent_agents_per_host: 5
```

Symphony 保持本机为调度中枢，Codex 在远程主机上执行，工作区也在远程创建。

### 场景三：按状态限制并发数

```yaml
agent:
  max_concurrent_agents: 10
  max_concurrent_agents_by_state:
    "In Progress": 5
    "Merging": 2
```

### 场景四：使用环境变量传递敏感值

```yaml
workspace:
  root: $SYMPHONY_WORKSPACE_ROOT

tracker:
  api_key: $LINEAR_API_KEY

codex:
  command: "$CODEX_BIN app-server"
```

---

## 日志查看

Symphony 默认将日志写到 `./log/` 目录（可通过 `--logs-root` 修改）：

```bash
# 查看实时日志
tail -f log/symphony.log

# 过滤某个工单的日志
grep "issue_identifier=MT-123" log/symphony.log

# 过滤某个 Codex 会话的日志
grep "session_id=thread-xxx" log/symphony.log
```

---

## 动态修改配置

Symphony 每秒检测 `WORKFLOW.md` 的变化，**无需重启**即可生效：

- 修改 `polling.interval_ms`：下一个 tick 开始生效
- 修改 `agent.max_concurrent_agents`：下一次调度决策生效
- 修改提示词模板：下一个新工单使用新模板
- 修改 `active_states`/`terminal_states`：下一次 reconciliation 生效

---

## 常见问题

**Q: 工单没有被自动处理？**

检查：
1. `LINEAR_API_KEY` 是否正确设置
2. `tracker.project_slug` 是否与 Linear URL 中的 slug 一致
3. 工单状态是否在 `active_states` 列表中
4. 终端日志是否有 `Missing WORKFLOW.md` 或 `Linear API token missing` 错误

**Q: 代理启动后立即失败？**

运行 `codex app-server --help` 确认 Codex 已正确安装并支持 app-server 模式。检查 `hooks.after_create` 脚本是否能在目标工作区目录成功执行。

**Q: 如何停止 Symphony？**

按 `Ctrl+C` 即可优雅退出。正在运行的代理任务会被终止，工作区文件保留以供下次续接。

**Q: 能否只运行特定工单，不监听整个看板？**

Symphony 目前不支持单工单模式，它始终监听整个项目。可以通过 Linear 的状态或标签来控制哪些工单进入活跃状态。

**Q: 工作区文件会自动清理吗？**

当 Issue 进入终止状态（`Done`、`Closed`、`Cancelled` 等）时，Symphony 会在下次 reconciliation 时自动删除对应工作区（先执行 `before_remove` 钩子）。

---

## 下一步

- 阅读 [`WORKFLOW.md`](../elixir/WORKFLOW.md) 了解完整的工作流提示词和状态机设计
- 阅读 [`SPEC.md`](../SPEC.md) 了解 Symphony 协议规范
- 阅读 [`docs/architecture-zh.md`](./architecture-zh.md) 了解内部架构设计
- 参考 [`.codex/skills/`](../.codex/skills/) 中的技能脚本，了解 `commit`、`push`、`land` 等工作流辅助工具
