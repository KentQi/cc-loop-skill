# 效能度量（自动化采集）

通过 `.cc-loop/metrics.jsonl` 实现零侵入度量采集，Agent 仅在生成报告时读取，平时完全不感知。

## 架构概览

```
闸门执行 / 任务流转 / 审查活动
        ↓
   metrics_cmd（shell 追加 JSONL）
        ↓
.cc-loop/metrics.jsonl
        ↓
   Agent 生成报告时读取 + 聚合
        ↓
   Markdown 报告（写入 state.md）
```

## 采集管道（metrics.jsonl）

### 文件位置

`.cc-loop/metrics.jsonl` — 每行一个 JSON 对象，追加写入，永不修改历史行。

### 事件类型

| type | 何时写入 | 关键字段 |
|------|---------|---------|
| `gate` | 闸门执行后 | gate, task_id, status, round, agent, duration_ms |
| `task_status` | 任务状态变更时 | task_id, from, to, round, agent |
| `review` | 审查完成后 | task_id, verdict, block_count, major_count, round, agent |
| `fix_attempt` | 修复尝试后 | task_id, gate, attempt_no, success, round, agent |
| `blocker` | 新增 blocker 时 | blocker_id, task_id, reason, round, agent |
| `blocker_resolve` | blocker 解决时 | blocker_id, task_id, resolution, round, agent |
| `escalation` | 升级触发时 | reason, from_agent, to_agent, round |

### 写入方式（方案 B：shell 确保格式）

在 `gates.yml` 中每个闸门配置增加 `metrics_cmd` 字段：

```yaml
gates:
  - id: lint
    name: ESLint 检查
    command: npm run lint
    autofix: npm run lint:fix
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"lint","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION}'}' >> .cc-loop/metrics.jsonl
```

闸门执行脚本在运行 `command` 后，无条件执行 `metrics_cmd` 追加一行记录。

### 示例数据

```jsonl
{"ts":"2026-06-24T10:00:00Z","type":"task_status","task_id":"TASK-003","from":"backlog","to":"claimed","round":5,"agent":"Developer"}
{"ts":"2026-06-24T10:01:00Z","type":"gate","gate":"lint","task_id":"TASK-003","status":"pass","round":5,"agent":"Developer","duration_ms":1200}
{"ts":"2026-06-24T10:02:00Z","type":"gate","gate":"unit-test","task_id":"TASK-003","status":"pass","round":5,"agent":"Developer","duration_ms":4500}
{"ts":"2026-06-24T10:05:00Z","type":"task_status","task_id":"TASK-003","from":"in-progress","to":"ready-for-review","round":5,"agent":"Developer"}
{"ts":"2026-06-24T10:10:00Z","type":"review","task_id":"TASK-003","verdict":"request_changes","block_count":1,"major_count":2,"round":6,"agent":"Reviewer"}
{"ts":"2026-06-24T10:15:00Z","type":"fix_attempt","task_id":"TASK-003","gate":"reviewer_feedback","attempt_no":1,"success":true,"round":7,"agent":"Developer"}
{"ts":"2026-06-24T10:20:00Z","type":"task_status","task_id":"TASK-003","from":"ready-for-review","to":"ready-for-test","round":7,"agent":"Reviewer"}
```

## 报告生成（Agent 聚合逻辑）

Agent 在 Sprint 回顾或 `/loop summary` 时读取 `metrics.jsonl`，按以下逻辑聚合：

### 聚合伪代码（Agent 参考）

```python
def generate_report(metrics_jsonl_path):
    """Agent 按此逻辑解析 metrics.jsonl 生成报告"""
    events = [json.loads(line) for line in open(metrics_jsonl_path)]

    # 1. 任务周期（Cycle Time）
    tasks = {}
    for e in events:
        if e["type"] == "task_status":
            tid = e["task_id"]
            if tid not in tasks:
                tasks[tid] = {"status_log": [], "gates": [], "reviews": []}
            tasks[tid]["status_log"].append(e)
    
    for tid, data in tasks.items():
        claimed = find_first(data["status_log"], to="claimed")
        done = find_first(data["status_log"], to="done")
        if claimed and done:
            data["cycle_rounds"] = done["round"] - claimed["round"]
        else:
            data["cycle_rounds"] = None  # 未完成

    # 2. 审查通过率
    reviews = [e for e in events if e["type"] == "review"]
    total = len(reviews)
    passed = len([r for r in reviews if r["verdict"] == "approve"])
    pass_rate = (passed / total * 100) if total else 0

    # 3. 阻塞频率
    blockers = [e for e in events if e["type"] == "blocker"]
    total_rounds = max(e["round"] for e in events) if events else 0
    blocker_freq = len(blockers) / total_rounds if total_rounds else 0

    # 4. 闸门健康度
    gates = {}
    for e in events:
        if e["type"] == "gate":
            gid = e["gate"]
            if gid not in gates:
                gates[gid] = {"total": 0, "pass": 0}
            gates[gid]["total"] += 1
            if e["status"] == "pass":
                gates[gid]["pass"] += 1

    return {
        "cycle_times": {tid: d["cycle_rounds"] for tid, d in tasks.items()},
        "review_pass_rate": pass_rate,
        "blocker_frequency": blocker_freq,
        "gate_health": {gid: g["pass"]/g["total"] for gid, g in gates.items()}
    }
```

### 报告输出格式

Agent 聚合后生成 Markdown 报告，写入 `state.md` 的 `## Metrics` 区域：

```markdown
## Metrics（Sprint [sprint-id]）

> 自动生成于 [timestamp]，基于 metrics.jsonl [N] 条记录

### 汇总
| 指标 | 值 | 等级 |
|------|-----|------|
| 总任务数 | [N] | — |
| 已完成 | [N] | — |
| 平均周期 | [N] 轮 | [🟢🟡🔴⚫] |
| 审查通过率 | [N]% | [🟢🟡🔴⚫] |
| 阻塞频率 | [N] | [🟢🟡🔴] |
| 总轮次 | [N] | — |

### 任务周期明细
| 任务 ID | 周期(轮) | 状态 | 等级 |
|---------|---------|------|------|
| TASK-001 | 3 | done | 🟡 |
| TASK-002 | 8 | done | 🔴 |
| TASK-003 | — | in-progress | — |

### 审查统计
| 审查者 | 审查数 | 通过 | 否决 | 通过率 | 平均 [major] |
|--------|--------|------|------|--------|-------------|
| Reviewer | 12 | 10 | 2 | 83% | 2.1 |

### 闸门健康度
| 闸门 | 检查 | 通过 | 自动修复 | 健康度 |
|------|------|------|---------|--------|
| lint | 24 | 22 | 2 | 🟢 92% |
| unit-test | 24 | 20 | N/A | 🟡 83% |
| type-check | 24 | 24 | N/A | 🟢 100% |

### 阻塞与升级
| 类型 | 次数 | 平均解决(轮) | 升级次数 |
|------|------|-------------|---------|
| 新增阻塞 | [N] | [N] | [N] |
| 死循环 Escalation | [N] | — | — |
```

## 分级标准

### 任务周期（Cycle Time）

| 等级 | 轮次 |
|------|------|
| 🟢 快速 | 1-2 |
| 🟡 正常 | 3-5 |
| 🔴 缓慢 | 6-10 |
| ⚫ 阻塞 | >10 |

### 审查通过率

| 等级 | 通过率 |
|------|--------|
| 🟢 优秀 | >85% |
| 🟡 正常 | 70-85% |
| 🔴 偏低 | 50-70% |
| ⚫ 危险 | <50% |

### 阻塞频率

| 等级 | 频率 |
|------|------|
| 🟢 低 | <0.2 |
| 🟡 中 | 0.2-0.5 |
| 🔴 高 | >0.5 |

## 度量驱动的回顾检查单

Coordinator 在每个 Sprint 结束时自动生成报告，并按以下检查单分析：

```
Sprint 回顾检查单：

□ 平均周期 > 5 轮？→ 分析慢任务根因，考虑拆分或优化流程
□ 审查通过率 < 70%？→ Developer 加强自测，Review 前确保 lint/test 通过
□ 阻塞频率 > 0.5？→ 根因分析，增加预防性设计或文档
□ 某闸门健康度 < 80%？→ 检查闸门阈值或 autofix 脚本
□ 死循环 Escalation > 1 次？→ 检查任务拆分粒度是否合适
□ 升级次数 > 2？→ 检查是否任务定义不清或技术债过重
```

## 轻量采集原则

1. **零侵入**：主循环中的 Agent 不感知度量系统，只有 shell 追加一行 JSONL
2. **Shell 保证格式**：`metrics_cmd` 使用 shell 命令确保 JSON 格式正确，不依赖 AI 拼接字符串
3. **按需聚合**：Agent 只在 Sprint 回顾或 `/loop summary` 时读取并聚合
4. **不追求完美数据**：度量辅助决策，不是 KPI 考核
5. **可归档**：metrics.jsonl 按月归档到 `.cc-loop/archive/metrics-YYYY-MM.jsonl`
