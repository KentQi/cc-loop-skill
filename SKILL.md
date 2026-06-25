---
name: cc-loop
description: >
  Claude Code 项目循环开发工作流管理技能。适用于需要长时间迭代、多 Agent 协作、
  质量闸门控制的复杂项目开发场景。当用户在 Claude Code 环境中需要：
  (1) 使用 /loop 或 /go 命令进行自主循环开发，
  (2) 管理多轮迭代中的项目状态和上下文持久化，
  (3) 设定质量闸门进行自动化检查，
  (4) 采用多 Agent 角色扮演（开发/测试/审查）进行并行工作，
  (5) 需要状态文件驱动的增量开发流程时——触发此技能。
  核心目标是通过结构化的状态文件和预定义流程，让 /loop 和 /go 在复杂项目中可控、可追踪、高质量地持续运转。
---

# cc-loop: Claude Code 循环开发工作流

通过状态文件驱动、质量闸门控制、多 Agent 协作，将 Claude Code 的 `/loop` 和 `/go` 转化为工业级项目开发流程。

## 核心概念

| 概念 | 说明 |
|------|------|
| **状态文件** | 单点真相源，记录项目背景、当前阶段、任务列表、质量检查结果。每轮循环自动更新。 |
| **质量闸门** | 预定义的通过标准，每轮迭代必须通过才能进入下一阶段。 |
| **Agent 角色** | 采用「7 + N」弹性角色团队：7个核心角色（必设）+ N个扩展角色（按需动态激活）。通过系统提示分配角色（架构师/开发者/测试者/审查者/协调者/产品/运维 + 扩展角色），并行执行任务。 |
| **循环模式** | `/loop` 用于多步骤持续迭代，`/go` 用于单任务自主执行。两者结合实现自动化流水线。 |
| **Change Plan** | 变更计划，Agent 修改代码前必须输出计划，用户确认后才执行（保守模式）。 |
| **Coordinator** | 协调者 Agent，专责状态文件维护、摘要更新、轮次管理。 |

## 启动检查单（必须完成）

**在任何 `/loop` 或 `/go` 启动前，必须先完成以下检查：**

1. **检查 `.cc-loop/` 目录**：如果不存在，创建并初始化配置文件
2. **检查 `goal.md`**：如果不存在或缺少验收标准，引导用户定义
3. **检查 `state.md`**：如果不存在，使用模板初始化
4. **检查 `gates.yml`**：如果不存在，使用项目类型默认配置

**初始化命令模板**：

```bash
# 创建目录结构
mkdir -p .cc-loop

# 创建 goal.md（含验收标准）
# 创建 state.md（使用 references/project-state-schema.md 模板）
# 创建 gates.yml（使用 references/quality-gates.md 默认配置）
```

**未完成初始化前，禁止启动 `/loop` 或 `/go`。**

## 状态引用策略（三级）

为避免上下文溢出，状态文件采用三级读取策略：

| 级别 | 内容 | 使用场景 |
|------|------|---------|
| **Summary** | 当前阶段、活跃任务、最新阻塞点、上轮摘要 | 默认，每轮 /loop 引用 |
| **Active Phase** | 当前阶段的完整任务清单和闸门状态 | 阶段切换时读取 |
| **Full State** | 完整历史记录 | 上下文丢失恢复、复盘时 |

Coordinator Agent 负责每轮结束后更新 Summary。

## 工作流总览

```
[初始化检查单] → 加载 Summary → 设定质量闸门 → 
[循环开始]
  Coordinator: 读取 Summary，确定当前轮次 Agent 分配
  Agent: 输出 Change Plan → [用户确认] → 执行 → 自测
  Coordinator: 收集结果，更新 state.md（追加历史 + 更新 Summary）
  质量闸门检查 → 通过? → 是: 继续 / 否: 修复循环
  检查死循环（3轮无推进?）→ 是: 触发 Escalation
[循环结束]
  所有阶段完成且质量通过 → 项目交付
```

## /loop 模式（主循环）

**适用场景**：功能开发、迭代改进、多轮协作任务。

**默认配置**：尝试 `--parallel` 多 Agent 并行，环境不支持时回退到角色轮转。

### 启动模板

```
/loop

引用：.cc-loop/goal.md, .cc-loop/SUMMARY（state.md 的摘要部分）

启动协议：
1. Coordinator 读取 SUMMARY，确定当前轮次和 Agent 分配
2. 活跃 Agent 先输出 Change Plan（保守模式）：
   - 要修改的文件
   - 修改原因
   - 预期影响
   - 依赖关系
3. [等待用户确认 Change Plan，或用户已添加 --yes 则自动继续]
4. Agent 执行变更
5. 运行自测（lint + unit-test）
6. Coordinator 更新 state.md：
   - 追加本轮历史记录（--- 分隔）
   - 更新 Summary（当前阶段、活跃任务、阻塞点）
   - 更新质量闸门状态表
7. 如全部闸门通过且所有任务 done → 完成
8. 如有失败 → 标记修复任务，继续循环
```

### 多 Agent 并行模式

```
/loop --parallel

引用：.cc-loop/goal.md, .cc-loop/SUMMARY

并行 Agent 配置（Coordinator 统一调度）：

Agent: Developer
- 从 SUMMARY 获取高优先级 pending 任务
- 输出 Change Plan → [用户确认] → 实现 + 单元测试
- 任务完成后标记 ready-for-review，@Reviewer

Agent: Reviewer  
- 监控 ready-for-review 任务
- 代码审查，输出 [block]/[major]/[minor] 分级意见
- 审查通过后标记 ready-for-test，@Tester
- 有 [block] 意见则退回 in-progress，@Developer

Agent: Tester
- 监控 ready-for-test 任务
- 执行集成测试和边界测试（破坏性思维）
- 发现 bug 则标记 in-progress + bug-type 标签，@Developer
- 测试通过则标记 done

Coordinator（独立）
- 每轮收集三方输出
- 更新 state.md 历史记录和 Summary
- 检测死循环（3轮无 done 任务 → Escalation）
- 冲突仲裁失败时记录 blockers

交叉验证规则：
- Reviewer 的 [block] 必须解决才能进入测试
- Tester 的阻塞 bug 优先于新功能开发
- 三方冲突时，Coordinator 请求 Architect 仲裁
- **扩展角色介入时**：SEC 的 [block] 与 Reviewer 同等处理；UX/UI 的 [block] 在涉及用户体验/视觉时优先于功能实现；BA 的 [block] 在涉及业务规则正确性时优先于技术方案
```

### 扩展角色激活

Coordinator 在读取 SUMMARY 时，根据项目特性判断需激活的扩展角色：

```
激活信号示例：
- 项目含支付/敏感数据 → 激活 SEC（安全工程师）
- 项目含复杂交互 → 激活 UX（UX 研究员）
- 项目含界面设计 → 激活 UI（UI 设计专家）
- 项目含 iOS/Android → 激活 MOBILE（移动端开发）
- 项目含 AI/大数据 → 激活 DATA（数据/算法工程师）
- 项目涉金融/医疗/政务 → 激活 BA（业务分析师）

激活后在 state.md 记录：active_roles: ["COORD", "ARCH", "DEV", "TEST", "REV", "DEVOPS", "PM", "SEC", "UX"]
```

扩展角色按各自触发条件介入（如 SEC 在 Architect 设计阶段进行威胁建模，UX 在 ready-for-review 时进行体验走查）。
```

## /go 模式（单点突破）

**严格限定场景**：
- Bug 修复（≤3 轮）
- 单点技术攻坚（架构决策、算法优化）
- 紧急热修复

**禁止使用场景**：
- 长周期功能开发（>3 轮）
- 需要多角色协作的复杂任务

### 启动模板

```
/go

引用：.cc-loop/goal.md, .cc-loop/SUMMARY

任务：[具体任务描述]
轮次限制：[1-3 轮]

执行协议：
1. 输出 Change Plan（修改文件、方案、预期影响）
2. [等待用户确认]
3. 执行变更
4. 运行对应质量闸门检查
5. 更新 state.md 中的任务状态
6. 如达到轮次限制仍未完成 → 标记 blocked，建议转为 /loop

约束：
- 遵循项目编码规范和架构模式
- 所有变更必须可测试
- 完成后 Coordinator 更新 SUMMARY
- 如遇到超出范围的依赖，记录在 blockers 中不要自行展开
- 严禁超过轮次限制继续执行
```

## 质量闸门控制

质量闸门是强制检查点，未通过时必须循环修复。

### 检查时机

1. **每轮 /loop 结束时**：自动运行当前阶段的所有闸门
2. **任务状态变更前**：该任务的专属闸门必须通过
3. **/go 任务完成后**：运行对应质量验证

### 自动修复与升级逻辑

```
闸门失败处理：

lint/format 失败：
  → 调用 autofix 脚本
  → 重试（max 2 次）
  → 仍失败 → 标记 blocked，转人工

unit-test 失败：
  → 判断失败类型：
    a) 测试代码错误（断言/模拟问题）→ Agent 自修
    b) 业务逻辑错误 → Agent 尝试修复（max 2 次）
  → 2 次后仍失败 → 标记 blocked，@Architect 或转人工

integration-test 失败：
  → 记录问题，不阻塞（warn 级别）
  → 必须在 state.md 记录待修复项

coverage 未达标：
  → 补充测试用例
  → 如确实无法达标，人工豁免并记录原因
```

完整闸门配置参考：references/quality-gates.md

## Coordinator 协调者（核心角色）

Coordinator 是 cc-loop 的运转中枢，负责状态维护、冲突仲裁、死循环检测。

### 职责

| 职责 | 说明 |
|------|------|
| 状态维护 | 每轮结束后更新 state.md 的历史记录和 Summary |
| Agent 调度 | 根据任务优先级和依赖关系分配 Agent |
| 死循环检测 | 监控 3 轮无状态推进时触发 Escalation |
| 冲突仲裁 | 多 Agent 意见冲突时初步协调，失败则升级 Architect |
| 上下文管理 | 管理状态文件的三级读取，防止上下文溢出 |

### 系统提示摘要

```
你是 Coordinator（协调者）。你是 cc-loop 的运转中枢，不直接编写业务代码，
专注于维护项目状态和协调 Agent 协作。

原则：
- 你是唯一有权直接修改 state.md Summary 区域的角色
- 每轮必须更新 Summary，保持其反映最新状态
- 死循环检测：连续 3 轮无任务推进 → 触发 Escalation
- 冲突时先协调，协调失败再升级 Architect
- 主动管理上下文：当历史记录过长时，建议归档旧轮次
```

完整 Coordinator 定义参考：references/agent-roles.md

## 任务状态机

所有任务必须遵循以下状态流转：

```
                    [Reviewer block]
                         ↑___|
backlog → claimed → in-progress → ready-for-review → reviewing
                                              |           |
                                          [pass]      [block]
                                              ↓           |
                                          ready-for-test  |
                                              |           |
                                          testing       |
                                         /        \      |
                                   [pass]        [fail]--|
                                       ↓
                                     done
```

### 状态定义

| 状态 | 含义 | 触发者 |
|------|------|--------|
| backlog | 待领取 | Coordinator |
| claimed | 已被某 Agent 领取 | Agent |
| in-progress | 开发/修复中 | Developer |
| ready-for-review | 待审查 | Developer → @Reviewer |
| reviewing | 审查中 | Reviewer |
| ready-for-test | 待测试 | Reviewer → @Tester |
| testing | 测试中 | Tester |
| done | 已完成 | Tester |
| blocked | 已阻塞 | 任何 Agent |

### 交接暗号

- **Developer → Reviewer**：`@Reviewer task-XXX ready-for-review`
- **Reviewer → Tester**：`@Tester task-XXX ready-for-test` 或 `@Developer task-XXX [block] reason`
- **Tester → Developer**：`@Developer task-XXX in-progress bug-type: [regression/missing-feature/edge-case]`
- **Tester → Done**：`task-XXX done`

## 关键规则

1. **状态文件优先**：任何决策前先读取最新 Summary，任何变更后 Coordinator 立即更新 Summary
2. **质量闸门不可跳过**：未通过的闸门必须修复或显式人工豁免
3. **/goal 必须引用**：每次 `/loop` 和 `/go` 开头必须引用 `.cc-loop/goal.md`
4. **Plan 先确认后执行**：默认保守模式，Change Plan 必须用户确认后才执行（支持 `--yes` 跳过）
5. **死循环自动检测**：Coordinator 检测 3 轮无推进 → 自动触发 `/go Architect` 复盘
6. **阻塞立即上报**：遇到无法自主解决的问题，立即记录 blockers 并停止等待人工决策
7. **角色不可混淆**：同一轮次中一个 Agent 只执行一个角色的职责
8. **历史不可覆盖**：状态文件追加更新，保留完整决策历史
9. **Coordinator 独家更新**：只有 Coordinator 能修改 Summary 区域
10. **/go 轮次限制**：`/go` 任务严格限制 1-3 轮，超时必须转 `/loop`

## 快速参考

| 命令 | 场景 | 模式 |
|------|------|------|
| `/loop` | 功能开发、迭代改进 | 顺序或并行 |
| `/loop --parallel` | 多 Agent 协作冲刺 | Developer+Reviewer+Tester 并行 |
| `/loop --yes` | 信任模式，Change Plan 自动确认 | 保守模式的快速版 |
| `/go` | Bug 修复、技术攻坚 | 单 Agent，1-3 轮限制 |
| `/go Architect` | 死循环 Escalation、架构决策 | 架构师角色介入 |

## 参考文档索引

- **项目状态格式**：references/project-state-schema.md（state.md 完整格式规范）
- **状态管理策略**：references/state-management.md（三级读取、Summary 协议）
- **质量闸门配置**：references/quality-gates.md（闸门配置、自动修复、升级逻辑）
- **Agent 角色定义**：references/agent-roles.md（「7 + N」弹性角色团队完整定义、激活条件、冲突解决机制）
- **提示词模板**：references/prompt-templates.md（核心7角色 + 扩展6角色系统提示模板、切换指令）
- **循环模式示例**：references/loop-patterns.md（7 种常用模式模板）
- **故障排除**：references/troubleshooting.md（4 个逃生舱、故障恢复流程）
- **效能度量**：references/metrics.md（度量指标定义、采集规范）
