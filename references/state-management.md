# 状态管理策略

状态文件管理是 cc-loop 的核心机制。本文档定义三级读取策略、Summary 维护协议和归档规则。

## 状态文件结构

```
.cc-loop/
├── goal.md            # 项目目标（较少变更）
├── state.md           # 状态文件（频繁追加更新）
│   ├── 元数据
│   ├── 项目背景（精简）
│   ├── 当前阶段
│   ├── 任务清单（活跃阶段）
│   ├── 质量闸门状态
│   ├── Agent 上下文
│   ├── SUMMARY 区域  ←── Coordinator 维护（每轮更新）
│   ├── 历史记录    ←── 追加（每轮 --- 分隔）
│   ├── Metrics 区域  ←── Sprint 回顾时写入报告
│   └── Blockers & Decisions
├── gates.yml          # 质量闸门配置
├── metrics.jsonl      # 🆕 自动化度量采集（shell 追加，Agent 不直接写入）
└── archive/           # 归档目录（历史轮次 + 旧 metrics）
```

## 三级读取策略

### Level 1: Summary（默认）

**使用场景**：每轮 /loop 默认引用
**内容**：当前阶段关键信息、活跃任务、最新阻塞点、上轮摘要
**维护者**：Coordinator（每轮结束后更新）
**大小控制**：不超过 50 行

Summary 格式：

```markdown
## SUMMARY（Coordinator 维护，不要手动编辑）

**当前阶段**：[phase] | 开始：[时间]
**活跃 Agent**：[角色列表]
**上轮轮次**：[Round N]

### 进行中任务
- [>] [task-id]: [描述] | owner: [Agent] | since: [时间]

### 待处理（Top 3）
- [ ] [task-id]: [描述] | priority: [P0-P3]

### 最新阻塞点
- [blocker-id]: [一句话描述] | since: [时间]

### 上轮结果
- 完成任务：[列表]
- 发现问题：[列表]
- 下轮计划：[一句话]

### 质量闸门
| 闸门 | 状态 | 最后检查 |
|------|------|---------|
| [id] | [status] | [时间] |
```

### Level 2: Active Phase（阶段切换时）

**使用场景**：进入新阶段时、长时间运行后上下文刷新
**内容**：当前阶段的完整任务清单、闸门配置、阶段目标
**引用方式**：明确指定 `引用：.cc-loop/state.md ## 当前阶段`

### Level 3: Full State（恢复时）

**使用场景**：
- 会话中断后恢复上下文
- 复盘时查看完整历史
- 解决复杂 blockers 时需要历史决策参考

**引用方式**：明确指定 `引用：.cc-loop/state.md`（完整文件）

**注意**：全量引用仅在必要时使用，平时优先使用 Summary。

## Coordinator 的 SUMMARY 维护协议

### 更新时机

| 时机 | 动作 |
|------|------|
| 每轮 /loop 结束时 | 更新 SUMMARY 区域 |
| 任务状态变更时 | 更新进行中/待处理任务列表 |
| 闸门状态变更时 | 更新质量闸门表 |
| 发现新阻塞点时 | 追加到最新阻塞点列表 |
| 阻塞点解决时 | 从活跃阻塞点移除，记录到已解决 |

### 更新规则

1. **覆盖更新**：SUMMARY 区域是 state.md 中唯一允许覆盖的部分（不是追加）
2. **精简原则**：SUMMARY 不超过 50 行，只保留关键信息
3. **活跃阻塞点最多 3 个**：超过时按优先级排序，低优先级移入历史
4. **待处理只显示 Top 3**：全部待处理任务在 Active Phase 区域查看

### 更新步骤

```
Coordinator 更新流程：
1. 读取当前 state.md 的 SUMMARY 区域
2. 收集本轮所有 Agent 的输出：
   - Developer: 完成的任务、修改的文件
   - Reviewer: 审查意见、[block]/[major]/[minor] 列表
   - Tester: 测试结果、发现的 bug
3. 更新 SUMMARY：
   - 更新进行中任务状态
   - 更新待处理 Top 3
   - 更新最新阻塞点
   - 填写上轮结果
   - 更新质量闸门状态
4. 在历史记录区域追加本轮详情（--- 分隔）
5. 如历史超过 20 轮 → 触发归档建议
```

## 归档规则

当历史记录超过 20 轮时：

```
Coordinator 归档流程：
1. 检测历史轮次 > 20
2. 创建归档文件：.cc-loop/archive/state-round-[start]-[end].md
3. 将 Round 1-10 的历史记录移动到归档文件
4. state.md 中保留 Round 11-20 的摘要，删除详细历史
5. 在 state.md 顶部添加归档索引：
   ## 归档索引
   - [rounds 1-10](archive/state-round-1-10.md)
```

### metrics.jsonl 管理

**初始化**（项目启动时）：
```bash
# 创建空文件（如果不存在）
touch .cc-loop/metrics.jsonl
# 确保文件可写
chmod +w .cc-loop/metrics.jsonl
```

**写入规则**：
- 只有 shell（通过 gates.yml 中的 `metrics_cmd`）直接写入
- Agent 永不直接写入 metrics.jsonl
- 所有事件追加到文件尾部，永不修改历史行
- JSONL 格式损坏时跳过损坏行，继续解析后续行

**归档规则**：
- metrics.jsonl 超过 10MB 或超过 10000 行时触发归档
- 归档命令：
  ```bash
  mv .cc-loop/metrics.jsonl .cc-loop/archive/metrics-$(date +%Y-%m).jsonl
  touch .cc-loop/metrics.jsonl
  ```
- 归档后更新 `.cc-loop/archive/index.md` 记录归档文件列表

## 状态文件防损坏协议

### 损坏检测

Coordinator 每轮检查：
- state.md 是否能正常解析
- SUMMARY 区域是否存在
- 任务 ID 是否唯一
- 状态值是否在有效枚举中

### 恢复流程

```
检测到 state.md 损坏：
1. 立即停止所有 Agent
2. 尝试从 git history 恢复：
   git show HEAD:.cc-loop/state.md > .cc-loop/state.md.backup
3. 如 git 恢复成功：
   - 用备份文件替换损坏文件
   - 从备份的 SUMMARY 恢复上下文
   - 标记恢复事件到历史记录
4. 如 git 恢复失败：
   - 从 goal.md 和归档文件重建最小状态
   - 标记为"重建状态，可能丢失近期历史"
   - 请求人工确认重建结果
```

## 上下文溢出防护

### 检测指标

Coordinator 监控：
- /loop 引用的总 token 数（估算）
- state.md 文件大小
- 当前会话的轮次

### 降级策略

```
Token 使用接近上限时：
1. 只引用 SUMMARY（不引用 Active Phase）
2. 历史记录归档（减少文件大小）
3. 简化 Change Plan 输出（只列文件和原因，省略详细影响分析）
4. 如仍超限：暂停循环，建议人工介入压缩状态文件
```
