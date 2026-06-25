# 故障排除指南

cc-loop 的逃生舱指令（Escape Hatches）和故障恢复流程。

## 故障类型速查

| 故障 | 现象 | 自动/手动 | 解决方案 |
|------|------|-----------|---------|
| 死循环 | 3 轮无状态推进 | **自动** | 触发 `/go Architect` 复盘 |
| 上下文溢出 | Token 接近上限 | 手动 | 降级到 Summary 模式 + 归档 |
| 角色混乱 | Agent 跨角色行为 | 手动 | 重置 Agent 上下文 |
| 状态损坏 | state.md 解析失败 | 手动 | git 恢复 或 重建 |
| 闸门死锁 | 同一闸门反复失败 | **自动** | 2 次后升级 Architect/人工 |
| 任务泄漏 | 任务长时间无响应 | 手动 | Coordinator 重新分配 |

## 逃生舱 1：死循环检测（自动触发）

### 触发条件

Coordinator 自动检测：
- 连续 3 轮无任务状态变更（无 done、无 in-progress → ready-for-review 等流转）
- 连续 3 轮同一 block 未解决
- 连续 3 轮质量闸门同一 fail 未修复

### 自动响应流程

```
Coordinator 检测到死循环：
1. 在 state.md 标记："Dead loop detected at Round N"
2. 自动触发：/go Architect

/go Architect
引用：.cc-loop/goal.md, .cc-loop/SUMMARY

任务：死循环复盘与突破

请分析：
1. 为什么最近 3 轮没有推进？（根因分析）
2. 当前 block 的技术/流程原因是什么？
3. 给出 2-3 个突破方案（含优缺点）
4. 推荐方案及理由

输出：
- 根因分析（3 句话以内）
- 突破方案（2-3 个）
- 推荐行动
- 预计需要几轮解决

限制：2 轮内必须给出明确结论
```

3. Architect 给出方案后，Coordinator 更新 SUMMARY 并重新调度
4. 如 Architect 2 轮内仍无法突破 → 标记为"需要人工决策"，暂停循环

## 逃生舱 2：上下文溢出（手动触发）

### 触发条件

- Claude 提示"上下文长度接近限制"
- /loop 响应明显变慢
- 状态文件超过 500 行

### 手动处理流程

```
用户或 Coordinator 发现上下文溢出：

步骤 1：降级读取
/loop
引用：.cc-loop/SUMMARY（不再引用完整 state.md）

步骤 2：立即归档
Coordinator 执行归档：
- 将历史记录中超过 10 轮的旧记录移入 .cc-loop/archive/
- 保留最近 10 轮

步骤 3：简化 Change Plan
要求 Agent 的 Change Plan 精简为：
- 修改文件（只列文件名）
- 一句话原因
- 不再写预期影响分析

步骤 4：如仍溢出
- 暂停 /loop
- 建议用户开启新会话
- 新会话启动时只引用 SUMMARY 恢复上下文
```

## 逃生舱 3：角色混乱（手动触发）

### 触发条件

- Developer 开始审查代码（应该是 Reviewer 的职责）
- Tester 开始写功能代码（应该是 Developer 的职责）
- Agent 同时执行多个角色的任务
- Agent 输出不符合当前角色的系统提示规范

### 手动处理流程

```
检测到角色混乱：

步骤 1：重置 Agent 上下文
/loop

引用：.cc-loop/SUMMARY

上下文重置：当前 Agent 角色混乱，执行重置。

当前正确角色分配：
- Developer: [task-XXX]
- Reviewer: 等待 ready-for-review
- Tester: 等待 ready-for-test

请所有 Agent 重新加载对应角色模板：
references/prompt-templates.md#[角色名]

步骤 2：重新加载角色模板
要求 Agent 明确声明：
"我当前的角色是 [Role]，我的职责是 [职责摘要]"

步骤 3：恢复任务状态
Coordinator 检查任务状态：
- 混乱期间完成的任务是否合规？
- 不合规的 → 回退到上一个合法状态重新执行
- 合规的 → 保留，继续流程
```

## 逃生舱 4：状态文件损坏（手动触发）

### 触发条件

- state.md 无法解析（格式错误、YAML  frontmatter 损坏）
- SUMMARY 区域丢失
- 任务 ID 冲突或状态值非法
- 文件被意外清空或截断

### 恢复流程

```
检测到状态文件损坏：

步骤 1：立即停止
所有 Agent 停止操作，不再写入 state.md

步骤 2：尝试 git 恢复
git show HEAD:.cc-loop/state.md > .cc-loop/state.md.backup
git show HEAD~1:.cc-loop/state.md > .cc-loop/state.md.backup-old

步骤 3：评估恢复点
比较 backup 和 backup-old：
- 如 backup 完整 → 用 backup 替换损坏文件
- 如 backup 也损坏 → 尝试 backup-old
- 如都损坏 → 进入重建流程

步骤 4A：git 恢复成功
cp .cc-loop/state.md.backup .cc-loop/state.md
在 SUMMARY 中添加："状态已从 git [commit] 恢复，可能丢失 [N] 轮的历史"
继续循环

步骤 4B：重建流程（git 恢复失败）
1. 从 goal.md 读取项目目标
2. 从归档目录读取历史任务（如有）
3. 扫描代码库当前状态：
   - 已完成的文件
   - 测试覆盖率
   - lint/type-check 状态
4. 重建最小状态文件：
   - 元数据（新建，标注"重建"）
   - 项目背景（从 goal.md 精简）
   - 当前阶段（根据代码库状态推断）
   - 任务清单（根据代码库 TODO/FIXME 扫描生成）
5. 请求人工确认重建结果
6. 确认后继续循环
```

## 其他故障处理

### 任务泄漏

现象：某任务被 claimed 后长时间无响应（超过 5 轮）

处理：
```
Coordinator 检测：
1. 标记该任务为 stalled
2. 释放 Agent 分配（允许其他 Agent 领取）
3. 原 Agent 后续回复时，要求先同步当前 SUMMARY
4. 如原 Agent 无法继续 → 任务重新进入 backlog
```

### 质量闸门死锁

现象：同一闸门连续失败，Agent 修复后仍失败

处理：
```
已内置在闸门逻辑中：
- lint/format: 2 次 autofix 后仍失败 → blocked → 人工
- unit-test: 2 次修复后仍失败 → @Architect
- integration: 记录 warn，不阻塞

如 Architect 也无法解决：
- 标记为"技术债务"
- 记录到 state.md 的 known-issues
- 人工豁免该闸门（标注原因）
- 后续 sprint 专项解决
```

### 多 Agent 冲突无法仲裁

现象：Coordinator + Architect 都无法协调冲突

处理：
```
1. Coordinator 记录冲突详情到 blockers
2. 暂停 /loop
3. 向用户呈现：
   - 冲突双方观点
   - 已尝试的协调方案
   - 各方案的优缺点
4. 等待人工决策
5. 决策后记录到 key-decisions，继续循环
```

## 预防性检查

Coordinator 每轮执行以下预防性检查：

```
□ state.md 文件大小 < 500 行？（否则触发归档建议）
□ SUMMARY 区域存在且格式正确？
□ 任务 ID 唯一，无冲突？
□ 所有任务状态值在合法枚举中？
□ 当前轮次与上轮轮次连续？
□ 活跃 blockers < 5 个？（否则建议优先解决）
```
