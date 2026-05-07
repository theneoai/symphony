# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Project Is

Symphony is a long-running orchestration service that polls a Linear board for work, creates isolated per-issue workspaces, and runs a Codex agent (`codex app-server`) inside each workspace. The reference implementation is in `elixir/`. The canonical behavior contract is `SPEC.md`. The Elixir implementation may extend the spec but must not conflict with it.

## Development Environment

Requires Elixir 1.19.x (OTP 28) managed via [mise](https://mise.jdx.dev/):

```bash
cd elixir
mise trust && mise install
mix setup       # fetch deps
mix build       # compile escript → bin/symphony
```

All commands below assume `cd elixir` first.

## Common Commands

| Task | Command |
|------|---------|
| Full CI gate | `make all` |
| Run all tests | `mix test` |
| Run a single test file | `mix test test/symphony_elixir/orchestrator_status_test.exs` |
| Run tests with coverage | `mix test --cover` |
| Format code | `mix format` |
| Check formatting | `mix format --check-formatted` |
| Lint (specs + credo) | `mix lint` |
| Check `@spec` coverage | `mix specs.check` |
| Dialyzer | `mix dialyzer` |
| Live e2e test | `export LINEAR_API_KEY=...; make e2e` |
| Validate PR body | `mix pr_body.check --file /path/to/pr_body.md` |
| Run Symphony | `./bin/symphony ./WORKFLOW.md [--port 4000] [--logs-root ./log]` |

`make all` runs: `setup → build → fmt-check → lint → coverage → dialyzer`.

## Architecture

### OTP Supervision Tree

`SymphonyElixir.Application` starts (in order):

1. `Phoenix.PubSub` — pub/sub bus for dashboard updates
2. `Task.Supervisor` (`:SymphonyElixir.TaskSupervisor`) — supervises per-issue agent tasks
3. `WorkflowStore` — GenServer that caches `WORKFLOW.md` and polls it for changes every 1 s
4. `Orchestrator` — single GenServer owning all dispatch state
5. `HttpServer` — optional Phoenix/Bandit server (enabled by `server.port` or `--port`)
6. `StatusDashboard` — terminal status surface

### Config / Workflow Pipeline

`WORKFLOW.md` is the single source of runtime configuration. It has YAML front matter (parsed by `WorkflowStore` → `Workflow` → `Config.Schema`) and a Liquid prompt template body. `Config.settings!()` is the typed accessor used everywhere. On invalid reload, `WorkflowStore` keeps the last known good config and logs the error.

### Orchestrator Loop

`Orchestrator` is the authority for all dispatch state: `running` (map), `claimed` (set), `retry_attempts` (map), token totals. On each tick it:
1. Reconciles running issues (stall detection + tracker state refresh)
2. Validates config (`Config.validate!()`)
3. Fetches candidate issues from Linear
4. Sorts by priority → age → identifier and dispatches eligible ones up to concurrency limits
5. Notifies the dashboard

Workers are `Task.Supervisor` children. When a task exits normally, the orchestrator schedules a 1-second continuation retry. Abnormal exits use exponential backoff (`10s * 2^(attempt-1)`, capped by `agent.max_retry_backoff_ms`).

### Agent Execution Path

`AgentRunner.run/3` → `Workspace.create_for_issue/2` → `before_run` hook → `Codex.AppServer` turn loop → `after_run` hook. Each session reuses the Codex thread for up to `agent.max_turns` back-to-back turns, refreshing issue state between turns. `AgentRunner` runs on a remote SSH host when `worker.ssh_hosts` is configured; the Codex subprocess is launched over SSH stdio, keeping the orchestrator as the single state authority.

### Codex AppServer Client

`Codex.AppServer` speaks JSON-RPC 2.0 over an Erlang port (stdio). It handles:
- Session init, thread start, turn start/stream
- Auto-approval when `approval_policy == "never"`
- `linear_graphql` dynamic tool calls (optional extension)
- Token accounting from `thread/tokenUsage/updated.tokenUsage.total` (absolute totals only — see `docs/token_accounting.md`)

### Workspace Safety

`PathSafety` enforces that every workspace path stays under `workspace.root`. `Workspace` sanitizes issue identifiers to `[A-Za-z0-9._-]` before creating directories. Codex is always launched with `cwd == workspace_path`.

### Web Dashboard (Optional)

When a port is configured, `HttpServer` starts a Phoenix LiveView dashboard at `/` and a JSON API at `/api/v1/state`, `/api/v1/<issue_identifier>`, and `POST /api/v1/refresh`. `ObservabilityApiController` and `DashboardLive` pull data via `Orchestrator.snapshot/0`.

## Code Conventions

- All public `def` functions in `lib/` must have an `@spec`. `@impl` callback implementations are exempt. Validate with `mix specs.check`.
- Do not add `@spec` to `defp` functions unless you choose to.
- Add config access through `SymphonyElixir.Config`, not ad-hoc env reads.
- Logs must include `issue_id=` and `issue_identifier=` for issue-related events, and `session_id=` for Codex session events. See `docs/logging.md`.
- Tracker kind `"memory"` is the in-process test double for the Linear adapter.

## PR Requirements

- PR body must follow `.github/pull_request_template.md` exactly.
- Behavior or config changes must update `README.md`, `elixir/README.md`, and `elixir/WORKFLOW.md` in the same PR.
- If implementation changes alter intended behavior, update `SPEC.md` in the same change.
