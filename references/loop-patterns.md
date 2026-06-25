# 循环模式示例集

cc-loop 的常用 /loop 和 /go 模式模板，按项目阶段和团队配置组织。

## 目录

- [模式 A：全新项目从零启动](#模式-a全新项目从零启动)
- [模式 B：功能迭代开发](#模式-b功能迭代开发)
- [模式 C：Bug 修复螺旋](#模式-cbug-修复螺旋)
- [模式 D：代码重构与偿还技术债](#模式-d代码重构与偿还技术债)
- [模式 E：多 Agent 并行冲刺](#模式-e多-agent-并行冲刺)
- [模式 F：代码审查流水线](#模式-f代码审查流水线)
- [模式 G：紧急热修复](#模式-g紧急热修复)
- [模式 H：安全专项审查](#模式-h安全专项审查)
- [模式 I：领域驱动开发](#模式-i领域驱动开发)
- [模式 J：多端并行开发](#模式-j多端并行开发)

---

## 模式 A：全新项目从零启动

适用场景：全新项目，从零搭建基础设施。

### 启动指令

```
/loop

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：完成项目基础设施搭建，达到可开发状态。

阶段顺序执行：

Phase 1 - 项目初始化：
- 初始化项目结构（package.json, tsconfig, 目录结构）
- 配置代码规范工具（ESLint, Prettier）
- 配置类型检查（TypeScript）
- 配置测试框架（Vitest/Jest）
- 质量闸门：lint, type-check 通过

Phase 2 - 核心架构：
- 设计并实现核心模块接口
- 配置依赖注入或服务定位器
- 设置日志和错误处理
- 质量闸门：unit-test 通过

Phase 3 - CI/CD 配置：
- 配置 GitHub Actions/GitLab CI
- 配置部署脚本
- 质量闸门：所有闸门通过

每完成一个 Phase，更新 state.md：
- 标记阶段完成
- 更新质量闸门状态
- 记录关键决策
```

### 状态文件初始任务

```markdown
## 任务清单

### 待处理 [backlog]
- [ ] task-001: 初始化项目结构和依赖 | priority: P0 | owner: DevOps | est: 1
- [ ] task-002: 配置 ESLint + Prettier | priority: P0 | owner: DevOps | est: 1
- [ ] task-003: 配置 TypeScript | priority: P0 | owner: DevOps | est: 1
- [ ] task-004: 配置测试框架 | priority: P0 | owner: DevOps | est: 1
- [ ] task-005: 设计核心模块架构 | priority: P0 | owner: Architect | est: 2
- [ ] task-006: 实现核心接口 | priority: P1 | owner: Developer | est: 2
- [ ] task-007: 配置 CI/CD 流水线 | priority: P1 | owner: DevOps | est: 1

### 扩展角色介入示例（按需激活）
- [ ] task-008: 威胁建模与安全基线配置 | priority: P0 | owner: SEC | est: 1 | 激活条件：项目涉及用户数据或 API
- [ ] task-009: 信息架构与交互流程设计 | priority: P0 | owner: UX | est: 2 | 激活条件：项目含前端界面
- [ ] task-010: 视觉规范与设计系统初始化 | priority: P1 | owner: UI | est: 2 | 激活条件：项目含自定义界面
- [ ] task-011: 移动端框架选型与初始化 | priority: P1 | owner: MOBILE | est: 1 | 激活条件：项目含 iOS/Android/小程序
- [ ] task-012: 数据架构与算法方案设计 | priority: P1 | owner: DATA | est: 2 | 激活条件：项目含 AI/大数据功能
- [ ] task-013: 领域模型与业务流程梳理 | priority: P0 | owner: BA | est: 2 | 激活条件：领域知识复杂（金融/医疗/政务）
```

---

## 模式 B：功能迭代开发

适用场景：已有项目，开发新功能。

### 启动指令

```
/loop --parallel

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：完成当前 sprint 的所有 backlog 任务。

并行 Agent 配置：

Agent: Developer
- 从 state.md backlog 按优先级领取任务
- 每个任务：实现 + 单元测试 + 自测
- 完成后标记 in-progress → done
- 将代码提交给 Reviewer

Agent: Tester
- 对 Developer 标记 done 的功能进行测试
- 执行集成测试和边界测试
- 发现缺陷记录到 state.md blockers
- 更新质量闸门中的测试相关项

Agent: Reviewer
- 审查 Developer 提交的每批代码
- 给出 [block]/[major]/[minor] 分级意见
- 确认修复后批准
- 更新审查状态到质量闸门

Agent: SEC（如激活）
- 在 Reviewer 审查阶段并行进行安全审查
- 对敏感数据、认证、授权相关代码进行威胁建模审查
- 发现安全问题时标记 `@Developer [task] sec-issue [block/major]`
- SEC 的 [block] 与 Reviewer 的 [block] 同等处理

Agent: UX（如激活）
- 对 Developer 提交的界面实现进行体验走查
- 检查信息架构、交互流程、异常状态覆盖
- 发现体验问题时标记 `@Developer [task] ux-issue [block/major/minor]`

Agent: UI（如激活）
- 对 Developer 提交的界面实现进行视觉走查
- 检查设计系统合规性、无障碍、状态完整性
- 发现视觉问题时标记 `@Developer [task] ui-issue [block/major/minor]`

同步规则：
- 每 3 轮进行一次三方同步（含扩展角色状态同步）
- Reviewer 的 [block] 意见必须解决
- SEC 的 [block] 意见（安全风险）优先于功能实现
- Tester 发现的阻塞 bug 优先于新功能开发
- 所有任务 done 且闸门通过 → 完成
```

---

## 模式 C：Bug 修复螺旋

适用场景：生产环境发现 bug，需要定位修复。

### 启动指令

```
/go

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：定位并修复以下 bug

Bug 描述：
[从 state.md 或用户输入获取]

约束：
- 不要引入回归问题
- 修复必须包含回归测试
- 如需架构调整，先咨询 Architect

执行步骤：
1. 分析复现路径
2. 定位根因（代码层面）
3. 设计最小修复方案
4. 实现修复 + 回归测试
5. 验证修复（单元测试 + 手动验证）
6. 更新 state.md：标记 bug 为已修复，记录根因和方案

如果 3 轮内无法定位根因：
- 记录已尝试的路径和排除的假设
- 升级到 Architect 分析
- 记录在 state.md blockers 中
```

### 后续 /loop 验证

```
/loop

引用：.cc-loop/state.md

目标：验证 bug 修复并确保无回归。

循环执行：
1. 运行回归测试套件
2. 检查相关模块的单元测试
3. 运行集成测试验证完整流程
4. 所有测试通过 → 标记修复完成
5. 有失败 → 分析并修复
```

---

## 模式 D：代码重构与偿还技术债

适用场景：代码质量下降，需要重构。

### 启动指令

```
/loop

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：安全地重构目标模块，不引入行为变更。

安全约束（必须遵守）：
1. 重构前确保目标模块有完整的单元测试覆盖
2. 每次只做一种重构（ extract method / rename / move 等）
3. 每次重构后立即运行测试
4. 测试通过才进行下一次重构

执行流程：
1. 评估当前测试覆盖率（如不足先补充测试）
2. 列出重构清单（优先级排序）
3. 逐项执行重构 + 测试
4. 更新 state.md 记录重构项和状态

禁止：
- 一次做多种重构
- 在重构时顺手加新功能
- 跳过测试直接重构
```

---

## 模式 E：多 Agent 并行冲刺

适用场景：时间紧迫，需要最大化并行度。

### 启动指令

```
/loop --parallel

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：在 N 轮内完成 Sprint 目标。

Agent 分配：

Agent: Developer-A（前端）
- 负责 UI 组件和交互实现
- 与 Developer-B 协商 API 接口

Agent: Developer-B（后端）
- 负责 API 实现和数据层
- 提供接口文档给 Developer-A

Agent: Tester
- 前后端联调测试
- API 契约测试
- 端到端场景测试

Agent: Reviewer
- 轮询审查两位 Developer 的代码
- 关注接口一致性

Agent: MOBILE（如激活，含移动端）
- 负责 iOS/Android/小程序 实现
- 与 Developer-A 协商跨平台组件规范
- 提供原生能力接口文档

Agent: DATA（如激活，含算法功能）
- 负责算法模型训练和部署
- 提供算法接口和数据契约
- 与 Developer-B 协商数据管道集成

协调规则：
- Developer-A 和 Developer-B 通过 state.md 同步接口约定
- MOBILE 与 Developer-A 通过 state.md 同步跨平台组件规范
- DATA 与 Developer-B 通过 state.md 同步数据管道和算法接口
- 每 2 轮 Reviewer 出一份综合审查报告（含 SEC、UX 扩展角色的审查意见）
- Tester 在第 3 轮开始介入测试（含 MOBILE 真机测试、DATA 算法验证）
- 最后 2 轮专注于集成和修复
```

---

## 模式 F：代码审查流水线

适用场景：需要深度代码审查，如重要模块上线前。

### 启动指令

```
/go

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：对以下代码进行深度审查

审查范围：
[文件/模块列表]

审查清单：
□ 正确性：逻辑正确，边界处理
□ 可读性：命名清晰，结构合理
□ 可测试性：易于测试，依赖可注入
□ 安全性：无注入、越权、泄露风险
□ 性能：无明显性能陷阱
□ 一致性：符合项目规范

输出要求：
- 每个文件的审查意见（按 [block]/[major]/[minor] 分级）
- 改进建议清单
- 总体评价（approve / request_changes）
- 更新 state.md 的 review 闸门状态
```

---

## 模式 G：紧急热修复

适用场景：生产紧急问题，需要快速修复上线。

### 启动指令

```
/go

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：紧急修复生产问题，最小改动快速上线。

原则：
- 只做最小必要修复，不重构
- 优先可验证性而非完美性
- 记录所有改动便于后续 review

流程：
1. 快速定位问题（允许临时日志/debug）
2. 实施最小修复
3. 写回归测试
4. 验证修复有效
5. 记录改动到 state.md
6. 标记后续需要完整 review 的 tech-debt 项

完成后创建后续任务：
- 完整代码审查
- 根因分析文档
- 预防措施
```

---

## 模式 H：安全专项审查

适用场景：项目涉及支付、敏感数据、合规要求，需要深度安全审查。

### 启动指令

```
/loop

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：完成安全专项审查，消除高危风险。

已激活扩展角色：SEC（安全工程师）

审查阶段：

Phase 1 - 威胁建模：
- SEC 与 Architect 共同进行 STRIDE 威胁建模
- 识别资产、威胁、现有控制措施
- 输出威胁建模报告
- 质量闸门：威胁建模报告通过 Architect 审查

Phase 2 - 代码安全审查：
- SEC 对所有涉及认证、授权、数据处理的代码进行安全审查
- Reviewer 同步进行常规代码审查
- SEC 的 [block] 意见与 Reviewer 的 [block] 同等处理
- 质量闸门：SEC 审查无 [block] 级别问题

Phase 3 - 合规检查：
- SEC 对照合规清单（GDPR/等保/PCI-DSS 等）逐条检查
- 输出合规检查报告
- 质量闸门：所有合规项通过或已记录豁免

Coordinator 调度：
- SEC 在 Phase 1 和 Phase 2 中作为核心审查者参与
- 每轮 SUMMARY 包含 SEC 的审查状态
- 安全 [block] 优先于功能实现
```

---

## 模式 I：领域驱动开发

适用场景：金融、医疗、政务等高度专业领域，业务逻辑复杂。

### 启动指令

```
/loop

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：在复杂业务领域中完成需求到代码的准确映射。

已激活扩展角色：BA（业务分析师）

阶段顺序：

Phase 1 - 领域建模：
- BA 与 PM 共同梳理业务规则，输出领域模型和术语表
- Architect 根据领域模型设计技术架构
- BA 审查架构对业务规则的承载能力
- 质量闸门：领域模型通过 BA 和 Architect 双重确认

Phase 2 - 业务规则实现：
- Developer 按业务规则清单实现功能
- BA 对业务逻辑实现进行走查（业务正确性审查）
- Reviewer 进行代码质量和一致性审查
- 质量闸门：BA 审查无 [block] 级别业务逻辑错误

Phase 3 - 业务场景测试：
- Tester 执行功能测试
- BA 提供边界业务场景和异常流程测试用例
- 所有业务规则必须有对应的测试用例覆盖
- 质量闸门：所有业务规则测试通过

Coordinator 调度：
- BA 在 Phase 1-3 中持续参与，作为业务正确性的最终守门人
- BA 的 [block] 在业务逻辑层面优先于 Reviewer 的 [block]
- 每轮 SUMMARY 包含 BA 的审查状态
```

---

## 模式 J：多端并行开发

适用场景：项目同时包含 Web、iOS、Android、小程序等多端。

### 启动指令

```
/loop --parallel

引用：.cc-loop/goal.md, .cc-loop/state.md

目标：在 N 轮内完成多端功能同步开发。

已激活扩展角色：MOBILE（移动端开发）、UI（UI 设计专家）

Agent 分配：

Agent: Developer（Web 前端）
- 负责 Web 端界面和交互实现
- 与 MOBILE 协商跨平台组件规范

Agent: MOBILE（移动端）
- 负责 iOS/Android/小程序 实现
- 与 Developer 协商跨平台组件规范
- 提供原生能力接口文档
- 关注平台规范（HIG / Material Design / 小程序设计规范）

Agent: Developer-B（后端）
- 负责 API 实现，支持多端数据需求
- 提供接口文档给 Developer 和 MOBILE

Agent: UI（UI 设计专家）
- 输出多端视觉规范（Web、iOS、Android、小程序各端适配）
- 对 Developer 和 MOBILE 的实现进行视觉走查
- 维护设计 token 和跨平台组件库规范

Agent: Reviewer
- 轮询审查 Web 端和移动端代码
- 关注跨平台接口一致性
- 关注各端平台规范合规性

Agent: Tester
- 多端联调测试
- 各端兼容性测试
- 端到端场景测试

协调规则：
- Developer 和 MOBILE 通过 state.md 同步跨平台组件规范
- UI 的视觉规范锁定后，Developer 和 MOBILE 才能开始实现
- 每 2 轮 Reviewer 出一份综合审查报告（含各端一致性检查）
- Tester 在第 3 轮开始介入多端联调测试
- 最后 2 轮专注于多端集成和修复
```

---

---

## 通用技巧

### 状态文件引用技巧

在 /loop 中高效使用状态文件：

```
# 启动时引用完整状态
/loop
引用：.cc-loop/goal.md, .cc-loop/state.md

# 如果状态文件过大，只引用关键部分
/loop
引用：.cc-loop/goal.md
上下文：当前阶段是 [phase]，进行中任务 [task-xxx]，质量闸门 [status]

# 恢复上下文时
/loop
引用：.cc-loop/state.md
继续从 Round [N] 开始，当前 Agent 角色：[role]
```

### 循环控制技巧

```
# 限定循环轮数
/loop
引用：.cc-loop/state.md
执行最多 5 轮，完成后总结进度和剩余工作

# 限定范围
/loop
引用：.cc-loop/state.md
只处理 state.md 中 priority: P0 的任务

# 人工检查点
/loop
引用：.cc-loop/state.md
每 3 轮暂停，等待人工确认后再继续
```

### 中断和恢复

```
# 中断时记录状态
当前已执行 X 轮，当前任务 [task] 进行到 [步骤]，
建议下轮从 [步骤] 继续

# 恢复时
/loop
引用：.cc-loop/state.md
从上轮中断点继续：[具体步骤]
```
