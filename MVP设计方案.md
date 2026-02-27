# AI Agent 协作与治理可视化平台 MVP 设计方案

## 1. MVP 目标与边界

### 1.1 目标（做什么）
- 提供一个本地运行的“异步人工审批网关”。
- Agent 可通过 MCP Tool 提交任务并查询结果。
- 人类可在 Web 页面完成审批、修改、拒绝。
- 支持 webhook 推送 + 轮询两种结果回传。
- 支持超时自动处理（通过/拒绝/失效）。

### 1.2 非目标（MVP 不做什么）
- 不做多租户、RBAC、组织管理。
- 不做复杂多节点工作流编排。
- 不做 Schema 动态渲染（先固定文本场景）。
- 不做企业级 BI 仪表盘（仅基础统计可选）。

## 2. 角色与核心用例

### 2.1 角色
- **Agent 开发者**：调用 `submit_review` 提交审批任务，再用 `get_review_status` 或 webhook 获取结果。
- **审核人员**：在 Web UI 中查看任务、编辑回复、审批/拒绝。
- **系统管理员（轻量）**：配置默认超时策略和回调重试参数。

### 2.2 核心用例
1. Agent 提交“待发评论”。
2. 审核员修改后通过。
3. 系统记录审计并回调 Agent。
4. Agent 拿到 `final_payload` 发真实请求。

## 3. 总体架构（MVP）

```text
[Agent/OpenClaw]
    | MCP Tools
    v
[MCP Server Layer] -----> [Review Service (FastAPI)]
                               |-- SQLite (tasks, reviews, settings, callbacks)
                               |-- Scheduler (超时巡检)
                               |-- Webhook Dispatcher (异步重试)
                               v
                          [Web UI (React + Tailwind)]
```

### 3.1 推荐技术栈
- 后端：**FastAPI + SQLAlchemy + Pydantic**
- MCP：官方 MCP Python SDK（Tool 暴露）
- DB：SQLite（WAL 模式）
- 前端：React + Vite + Tailwind
- 任务调度：APScheduler（每分钟巡检）

## 4. 状态机设计

### 4.1 任务状态
- `pending`：待审批
- `approved`：已通过（含“修改后通过”）
- `rejected`：已拒绝
- `timeout`：超时（根据策略自动落状态）

### 4.2 状态流转
- `pending -> approved`
- `pending -> rejected`
- `pending -> timeout`
- 终态不可逆（MVP 先不支持撤销）

### 4.3 补充字段
- `decision_type`: `approve` / `edit_and_approve` / `reject` / `auto_timeout`
- `final_payload`: 人工最终确认版本（拒绝时可为空）
- `feedback`: 审核意见（可选）

## 5. 数据模型（SQLite）

### 5.1 `tasks`
- `id` (uuid, pk)
- `scenario` (text, indexed)
- `status` (text, indexed)
- `payload_json` (text)
- `final_payload_json` (text, nullable)
- `feedback` (text, nullable)
- `webhook_url` (text, nullable)
- `timeout_at` (datetime, indexed)
- `timeout_action` (text: `auto_approve|auto_reject|mark_timeout`)
- `created_at`, `updated_at`, `decided_at` (datetime)

### 5.2 `task_reviews`
- `id` (pk)
- `task_id` (fk)
- `action` (`approve|edit_and_approve|reject|auto_timeout`)
- `editor_note` (text)
- `before_payload_json` (text)
- `after_payload_json` (text)
- `reviewer` (text, MVP 可固定 `local_user`)
- `created_at`

### 5.3 `settings`
- `key` (pk)
- `value` (text/json)

### 5.4 `callback_logs`
- `id` (pk)
- `task_id` (fk)
- `target_url`
- `request_body_json`
- `status_code` (nullable)
- `result` (`success|failed`)
- `retry_count`
- `next_retry_at` (nullable)
- `created_at`

## 6. MCP Tool 契约（MVP）

### 6.1 `submit_review`
**入参**
- `scenario: string`
- `payload: object`
- `webhook_url?: string`
- `timeout_config?: { timeout_minutes?: number, timeout_action?: "auto_approve"|"auto_reject"|"mark_timeout" }`

**返回**
- `task_id: string`
- `status: "pending"`
- `timeout_at: string(ISO8601)`

**校验要点**
- `scenario` 非空
- `payload` 必须是 JSON object
- `webhook_url` 若传入则做 URL 基础校验

### 6.2 `get_review_status`
**入参**
- `task_id: string`

**返回**
- `task_id`
- `status: pending|approved|rejected|timeout`
- `final_payload: object|null`
- `feedback: string|null`
- `updated_at`

## 7. Web UI 设计

### 7.1 页面 A：Task Dashboard
- 列：Task ID、Scenario、Status、提交时间、超时时间、是否有 webhook
- 筛选：`scenario`、`status`
- 搜索：按 task_id
- 点击进入详情

### 7.2 页面 B：Review Workspace
- 左侧：原始 `payload`（JSON 查看）
- 右侧：可编辑文本区域（针对 `payload.text`）
- 操作按钮：
  - 【通过】（不改内容）
  - 【修改并通过】（保存编辑后通过）
  - 【拒绝】（可填写 feedback）
- 底部：审计记录时间线（MVP 可简化成最近一条）

### 7.3 页面 C：Settings
- 默认超时时间（分钟/小时）
- 默认超时动作（自动通过 / 自动拒绝 / 标记超时）
- webhook 重试次数、重试间隔

## 8. 后端核心流程

1. `submit_review` 创建任务（`pending`）。
2. 若带单次超时配置，覆盖全局默认。
3. 审核员在 UI 决策写入 `tasks + task_reviews`。
4. 若存在 `webhook_url`，异步 POST 回调。
5. 回调失败记录 `callback_logs` 并按策略重试。
6. 定时巡检处理过期 `pending` 任务，写入 `timeout` 或自动通过/拒绝结果。

### 8.1 Webhook payload 示例
```json
{
  "task_id": "xxx",
  "scenario": "xhs_comment",
  "status": "approved",
  "final_payload": {"url":"...","text":"人工修改后的回复"},
  "feedback": "语气已调整",
  "decided_at": "2026-02-27T10:00:00Z"
}
```

## 9. 安全与治理（MVP 最低限）

- 本地部署默认仅监听 `127.0.0.1`（若需局域网再开放）。
- 简单鉴权：UI 使用单密码（环境变量）。
- 输入输出全量日志（涉及敏感信息可做字段脱敏）。
- webhook 白名单域名（可选，建议启用）。

## 10. 里程碑建议（2~4 周）

- **Week 1**：数据模型、MCP 两个工具、任务状态流转。
- **Week 2**：Dashboard + Review Workspace + Settings。
- **Week 3**：Webhook 异步重试、超时巡检、基础审计。
- **Week 4（可选）**：体验打磨、导出 JSON 数据（为后续数据飞轮铺路）。

## 11. MVP 验收标准（可量化）

- Agent 可在 5 分钟内接入并提交成功。
- 审核闭环成功率 > 99%（本地单机）。
- 任务状态一致性：终态无脏写、无回滚。
- webhook 成功率可观测（失败可重试可追踪）。
- 超时任务 1 分钟内被巡检处理。
