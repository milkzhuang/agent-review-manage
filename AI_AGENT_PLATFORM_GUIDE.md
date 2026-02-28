# AI Agent Review Gateway 复现指南（Agent 执行版）

## 1. 文档目标与适用范围

本指南用于指导任意 Coding Agent 在最短路径内复现当前项目（`xhs_management`）的 MVP 能力，目标是跑通以下闭环：

1. Agent 通过 MCP 提交待审任务（`submit_review`）。
2. 人类在 Web UI 进行审批（通过/拒绝）。
3. Agent 通过轮询（`get_review_status`）或 Webhook 获取终态结果。
4. 系统支持超时自动处理与回调重试。

本指南以当前代码实现为准，而非理想设计稿。

---

## 2. 需求背景（为什么要做）

在 AI Agent 自动执行社交/客服/营销动作时，纯自动流程存在内容风险（幻觉、不当表达、合规问题）；纯人工又无法规模化。

项目定位是一个本地可部署的 Human-in-the-loop 审批网关：

- 对 Agent：提供标准 MCP Tool（低接入成本）。
- 对人工审核员：提供轻量 Web 审批台（低操作成本）。
- 对系统治理：提供超时策略、审计日志、幂等防重、回调重试（可运维）。

---

## 3. MVP 功能边界

### 3.1 In Scope（已实现）

- MCP Tools：
  - `submit_review`
  - `get_review_status`
- MCP 双传输：
  - `stdio`
  - HTTP/SSE（`/mcp`）
- Web UI：
  - 任务看板（筛选 + 汇总）
  - 审查工作台（通过/拒绝）
  - 系统设置（默认超时秒数、超时动作）
- 调度：
  - 超时巡检（60 秒）
  - Webhook 回调投递（2 秒轮询）
  - 失败重试（5s/30s/120s）
- 存储：
  - SQLite（WAL 模式）
- 安全：
  - 默认仅本地监听
  - 可选 Basic Auth（`ADMIN_PASSWORD`）

### 3.2 Out of Scope（MVP 未做）

- 多租户 / RBAC / SSO
- 多节点工作流编排
- Schema-driven 动态表单渲染
- 企业级审计看板与 BI 报表

---

## 4. 技术选型（当前真实实现）

- 运行时：Node.js `>=22`（使用内置 `node:sqlite`）
- 包管理：pnpm `10.x`
- 语言：TypeScript（Monorepo）
- 后端 HTTP：Fastify
- MCP：`@modelcontextprotocol/sdk`
- 校验：Zod
- 数据库：SQLite（单文件）
- 前端：原生 HTML + CSS + JS（无 React/Vue）
- 测试：Vitest

---

## 5. 代码结构总览

```text
xhs_management/
├── apps/server
│   ├── src
│   │   ├── bin/                 # mcp-http / mcp-stdio 入口
│   │   ├── api/http-app.ts      # REST + /mcp + 静态资源
│   │   ├── mcp/                 # MCP server + tool 注册
│   │   ├── services/            # 核心业务：任务、设置、回调
│   │   ├── scheduler/           # 超时与回调调度
│   │   ├── db/                  # SQLite schema 与初始化
│   │   └── tests/               # 集成/服务测试
│   └── public/                  # UI 静态页面
├── packages/shared/src/types.ts # 前后端共享类型
├── scripts/example-agent.ts     # 最小 Agent 示例
└── scripts/webhook-receiver.ts  # 最小回调接收器
```

---

## 6. 核心业务流与状态机

### 6.1 主流程图

```mermaid
flowchart LR
    A["Agent 调用 submit_review"] --> B["ReviewService 落库为 pending"]
    B --> C["Web UI 人工审批 或 Timeout Scheduler 自动处理"]
    C --> D["状态进入 approved/rejected/timeout"]
    D --> E["若存在 webhook_url -> callback_status=pending"]
    E --> F["WebhookDispatcher 投递回调 + 重试"]
    D --> G["Agent 轮询 get_review_status 获取终态"]
```

### 6.2 状态机

```mermaid
stateDiagram-v2
    [*] --> pending
    pending --> approved: 人工通过 / 超时自动通过
    pending --> rejected: 人工拒绝 / 超时自动拒绝
    pending --> timeout: 超时标记失效
    approved --> [*]
    rejected --> [*]
    timeout --> [*]
```

---

## 7. 数据模型（SQLite）

核心表：

1. `review_task`
2. `review_action_log`
3. `system_setting`

关键设计点：

- 幂等键：`(agent_id, client_request_id)` 唯一索引（当 `client_request_id` 非空）
- 并发安全：状态迁移 SQL 使用 `WHERE task_id=? AND status='pending'`
- 回调字段：`callback_status/callback_attempts/callback_next_retry_at` 等
- 索引：按 `status+expire_at`、`scenario+status`、回调重试时间建立索引

---

## 8. 模块级核心实现路径（含伪代码）

### 8.1 提交任务 `submit_review`

位置：
- `apps/server/src/mcp/register-review-tools.ts`
- `apps/server/src/services/review-service.ts`
- `apps/server/src/services/scenarios.ts`

伪代码：

```pseudo
submit_review(args):
  validate agent_id/scenario
  payload = validateScenarioPayload(scenario, payload)
  if client_request_id exists:
    if existing task(agent_id, client_request_id):
      return existing task summary

  timeout = merge(input.timeout_config, settings defaults)
  webhook_url = normalize or null
  create task_id + expire_at

  transaction:
    insert review_task(status='pending')
    insert review_action_log(action='submit')

  return {task_id, status='pending', expire_at}
```

实现特性：

- 场景注册表当前仅支持 `xhs_comment`。
- `payload` 必须包含合法 `url` + 非空 `text`。
- `review_text_path` 解析失败直接 400（`INVALID_REVIEW_TEXT`）。

### 8.2 审批通过/拒绝

位置：
- `apps/server/src/api/http-app.ts`
- `apps/server/src/services/review-service.ts`

伪代码：

```pseudo
approve(task_id, input):
  row = find task
  if row.status != pending -> 409

  if approve_mode == pass:
    final_payload = original payload
  else if approve_mode == edit_pass:
    final_payload = validate scenario payload
  else:
    400

  transitionPendingTask(to='approved', final_payload, feedback, reviewer_id)
  return task_detail
```

```pseudo
reject(task_id, input):
  row = find task
  if row.status != pending -> 409
  transitionPendingTask(to='rejected', final_payload=null, ...)
  return task_detail
```

注意：

- UI 已移除“编辑后通过”入口，但后端仍兼容 `edit_pass`（带弃用告警）。

### 8.3 超时巡检

位置：
- `apps/server/src/scheduler/runtime-scheduler.ts`
- `apps/server/src/services/review-service.ts`

伪代码：

```pseudo
every 60s:
  rows = select pending where expire_at <= now limit 100
  for each row:
    if timeout_action == auto_approve: transition to approved
    if timeout_action == auto_reject: transition to rejected
    if timeout_action == mark_timeout: transition to timeout
```

### 8.4 Webhook 投递与重试

位置：
- `apps/server/src/services/webhook-dispatcher.ts`
- `apps/server/src/services/review-service.ts`

伪代码：

```pseudo
every 2s:
  jobs = list callback_status='pending' and callback_next_retry_at <= now
  for each job:
    event_id = existing or new
    body = review.completed event
    headers include x-event-id/x-event-timestamp
    if webhook_secret exists:
      x-signature = HMAC_SHA256(ts + "." + body)

    POST webhook_url (10s timeout)
    if 2xx:
      mark callback success
    else:
      attempts += 1
      if attempts < 4:
        next_retry_at = now + [5,30,120][attempt-1]
        keep pending
      else:
        mark failed
```

### 8.5 MCP HTTP 会话管理

位置：
- `apps/server/src/api/http-app.ts`

关键逻辑：

- `POST /mcp`：
  - 首次请求必须是 initialize（无 session id）
  - 服务端创建 `StreamableHTTPServerTransport` 并绑定 `mcp-session-id`
- 后续 `POST/GET/DELETE /mcp`：
  - 必须携带有效 `mcp-session-id`
- 无效会话：`400 INVALID_MCP_SESSION`

---

## 9. API 合约（复现时必须保持）

### 9.1 MCP Tools

1. `submit_review`
- 入参：`agent_id`, `scenario`, `payload`, 可选 `webhook_url/timeout_config/client_request_id`
- 出参：`task_id`, `status`, `expire_at`

2. `get_review_status`
- 入参：`task_id`
- 出参：`task_id`, `status`, `final_payload`, `feedback`, `reviewed_at`

### 9.2 REST

- `GET /api/health`
- `GET /api/tasks`
- `GET /api/tasks/:task_id`
- `POST /api/tasks/:task_id/approve`
- `POST /api/tasks/:task_id/reject`
- `GET /api/settings`
- `PUT /api/settings`

### 9.3 常见错误码

- `INVALID_SCENARIO`
- `INVALID_PAYLOAD`
- `INVALID_WEBHOOK_URL`
- `INVALID_MCP_REQUEST`
- `INVALID_MCP_SESSION`
- `TASK_ALREADY_PROCESSED`（409）

---

## 10. UI 复现规范（当前版本）

### 10.1 信息架构

- 侧边导航三标签：
  - 总览（Dashboard）
  - 审查工作台（Workspace）
  - 系统设置（Settings）

### 10.2 Dashboard

- 顶部四个状态统计卡片（pending/approved/rejected/timeout）
- 筛选区（status/scenario/task_id）
- 桌面端表格 + 移动端卡片列表（同数据源）

### 10.3 Workspace

- 左侧任务列表（全部/待处理 + task_id 搜索）
- 右侧详情：
  - 审查文本（`review_text`）
  - 上下文信息（`context_info`）
  - 原始 payload JSON
  - 反馈输入 + reviewer_id
  - 操作按钮：`直接通过`、`拒绝`

### 10.4 Settings

- `default_timeout_seconds`
- `default_timeout_action`

### 10.5 响应式策略

- `<1279px`：简化表格列、栅格压缩
- `<767px`：侧边栏横向、隐藏表格改卡片、审批按钮固定底部

---

## 11. Agent 复现执行手册（最短路径）

### 11.1 环境准备

```bash
cd /Users/zhuangmulin/xhs_management
node -v   # >= 22
pnpm -v   # >= 10
pnpm install
```

复制环境变量：

```bash
cp .env.example .env
```

### 11.2 启动方式（推荐 HTTP）

```bash
pnpm mcp:http
```

验收：

- Web UI: `http://127.0.0.1:8787/`
- MCP: `http://127.0.0.1:8787/mcp`
- Health: `GET /api/health` 返回 `status=ok`

### 11.3 闭环烟雾测试（自动审批）

```bash
pnpm example:agent:http -- --auto-approve-after-ms=2000
```

期望：

- 控制台输出 `submit_review -> task_id=...`
- 状态从 `pending` 变为 `approved`（终态）

### 11.4 Webhook 闭环测试（可选）

终端 A：

```bash
pnpm example:webhook-receiver
```

终端 B：

```bash
pnpm example:agent:http -- --use-webhook --webhook-url=http://127.0.0.1:9000/review-callback --auto-approve-after-ms=2000
```

期望：

- receiver 收到 `review.completed` 事件
- 包含 `x-event-id`、`x-event-timestamp`（启用 secret 时含 `x-signature`）

### 11.5 stdio 复现（本地 Agent 首选）

```bash
pnpm example:agent:stdio -- --auto-approve-after-ms=2000
```

---

## 12. 从零重建时的实现顺序（供新 Agent 编码）

按以下顺序落地最稳：

1. 共享类型与状态常量（`packages/shared`）
2. SQLite schema + DB 初始化（WAL + 索引 + 幂等唯一键）
3. `SettingsService`（默认策略读写）
4. `Scenario` 注册与 payload 校验
5. `ReviewService`（submit/query/approve/reject/timeout transition）
6. MCP server + tools
7. Fastify REST + `/mcp` transport session
8. WebhookDispatcher + RuntimeScheduler
9. Web UI（看板 + 工作台 + 设置）
10. 示例 Agent + webhook receiver + 测试

完成定义：

- MCP submit/status 可用
- 人工审批可改变状态
- 超时策略生效
- 回调可发送且可重试
- UI 全链路可操作

---

## 13. 安全与运维约束

- 非本地 HOST 必须配置 `ADMIN_PASSWORD`，否则服务拒绝启动。
- Basic Auth 作用于 `/api/*` 与 `/mcp`（`/api/health` 除外）。
- 推荐仅监听 `127.0.0.1`。
- 回调签名建议启用 `WEBHOOK_SECRET`。
- 生产化前需补：TLS、审计留存、RBAC、限流与告警。

---

## 14. 复现验收清单（Agent 可逐条打勾）

1. 能启动 `pnpm mcp:http` 且健康检查通过。
2. MCP 能发现并调用 `submit_review`、`get_review_status`。
3. `submit_review` 返回 `task_id` 且数据库有 `pending` 记录。
4. Web UI 能查看任务并执行通过/拒绝。
5. `get_review_status` 能返回终态（approved/rejected/timeout）。
6. `client_request_id` 重复提交不生成新任务（幂等）。
7. 超时任务会按策略自动收敛至终态。
8. webhook 失败会按 5s/30s/120s 重试，最终失败态可见。
9. 移动端布局可用（任务卡片 + 底部操作条）。

---

## 15. 当前已知实现差异（避免复现歧义）

1. 设计文档曾提到 React/Tailwind，但当前实际 UI 为原生静态文件（`public` 目录）。
2. 设计稿包含“修改后通过”主路径，当前 UI 只暴露“直接通过/拒绝”；后端仍兼容 `edit_pass` 以保持向后兼容。
3. 运行时 Node 要求为 `>=22`（依赖 `node:sqlite`）。

