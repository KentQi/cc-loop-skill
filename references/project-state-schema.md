# 项目状态文件格式规范

状态文件（`.cc-loop/state.md`）是 cc-loop 的单点真相源，所有 Agent 共享此文件进行协作。

## 文件结构

```markdown
# Project State: [项目名称]

## 元数据
- created: [ISO 8601 时间戳]
- updated: [ISO 8601 时间戳]
- version: [状态文件版本号，从 1 开始递增]
- status: [active|paused|blocked|completed]
- current_sprint: [当前迭代标识]

## 项目背景
[项目目标、技术栈、约束条件的摘要。当 goal.md 内容较多时，此处放精简版]

## 当前阶段
phase: [当前所处阶段名称]
phase_start: [阶段开始时间]
phase_goal: [本阶段目标一句话描述]

## 任务清单

### 待处理 [backlog]
- [ ] task-001: [任务描述] | priority: [P0-P3] | owner: [Agent角色] | est: [预估轮次]
- [ ] task-002: [任务描述] | priority: P1 | owner: Developer | est: 2

### 进行中 [in-progress]
- [>] task-003: [任务描述] | started: [时间戳] | owner: Developer | blocker: [如有]

### 已完成 [done]
- [x] task-000: [任务描述] | completed: [时间戳] | owner: Developer | result: [结果摘要]

### 已阻塞 [blocked]
- [!] task-004: [任务描述] | blocked_since: [时间戳] | reason: [阻塞原因] | needs: [需要的人工决策]

## 质量闸门状态

| 闸门 | 状态 | 最后检查 | 结果 |
|------|------|---------|------|
| lint | pass/fail | timestamp | details |
| unit-test | pass/fail | timestamp | X/Y passed |
| integration | pending/pass/fail | timestamp | details |
| type-check | pass/fail | timestamp | details |
| coverage | pass/fail | timestamp | Z% |
| review | pending/pass/fail | timestamp | N issues |

## Agent 上下文

### 当前活跃 Agent
- role: [当前主导角色]
- context: [该 Agent 的上下文摘要，每轮更新]

### 上轮摘要
- round: [轮次编号]
- timestamp: [时间戳]
- agent: [执行 Agent]
- completed: [完成的任务]
- issues: [发现的问题]
- next: [下轮计划]

## 历史记录

---
### Round N [timestamp]
[本轮详细变更记录，包含所有 Agent 的操作、决策、问题]

### Round N-1 [timestamp]
[历史记录...]
---

## Blockers & Decisions

### 活跃阻塞项
- blocker-001: [描述] | since: [时间] | proposed_solution: [建议方案] | decision_pending: [true/false]

### 已解决
- ~~blocker-000: [描述] | resolved: [时间] | resolution: [解决方案]~~

### 关键决策记录
- decision-001: [决策内容] | made_by: [人工/Agent] | at: [时间] | rationale: [决策理由]
```

## 初始化示例

```markdown
# Project State: API Gateway Service

## 元数据
- created: 2026-06-24T10:00:00Z
- updated: 2026-06-24T10:00:00Z
- version: 1
- status: active
- current_sprint: sprint-01

## 项目背景
构建 API 网关服务，提供路由、鉴权、限流、日志功能。
技术栈：Node.js + Express + Redis + PostgreSQL。
约束：单服务部署，响应时间 < 100ms。

## 当前阶段
phase: architecture
phase_start: 2026-06-24T10:00:00Z
phase_goal: 完成服务架构设计和核心模块接口定义

## 任务清单

### 待处理 [backlog]
- [ ] task-001: 设计路由模块接口 | priority: P0 | owner: Architect | est: 1
- [ ] task-002: 设计鉴权中间件接口 | priority: P0 | owner: Architect | est: 1
- [ ] task-003: 设计限流策略接口 | priority: P1 | owner: Architect | est: 1
- [ ] task-004: 设计日志收集接口 | priority: P1 | owner: Architect | est: 1

### 进行中 [in-progress]
_空_

### 已完成 [done]
_空_

### 已阻塞 [blocked]
_空_

## 质量闸门状态

| 闸门 | 状态 | 最后检查 | 结果 |
|------|------|---------|------|
| lint | pending | - | - |
| unit-test | pending | - | - |
| integration | pending | - | - |
| type-check | pending | - | - |
| coverage | pending | - | - |
| review | pending | - | - |

## Agent 上下文

### 当前活跃 Agent
- role: Architect
- context: 刚初始化项目，准备开始架构设计阶段

### 上轮摘要
_空_

## 历史记录
---
_空_
---

## Blockers & Decisions

### 活跃阻塞项
_空_

### 已解决
_空_

### 关键决策记录
_空_
```

## 更新规则

1. **追加而非覆盖**：所有历史记录追加到文件尾部，用 `---` 分隔轮次
2. **元数据必更新**：每次修改后更新 `updated` 和 `version` 字段
3. **任务状态流转**：backlog → in-progress → done/blocked，单向流转
4. **闸门实时记录**：每次质量检查后更新闸门状态表
5. **阻塞及时上报**：发现阻塞立即写入 blockers，不要等待
