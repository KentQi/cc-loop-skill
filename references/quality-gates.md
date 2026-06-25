# 质量闸门配置手册

质量闸门是强制性的检查点，控制项目从阶段 A 进入阶段 B。

## 闸门执行原则

1. **串行执行**：有依赖的闸门串行（如 build 依赖 lint/type-check 通过）
2. **自动修复优先**：lint/format 类问题先尝试 autofix，不阻塞人工
3. **失败分级处理**：block 级必须修复，warn 级记录继续
4. **升级机制**：业务逻辑错误 2 次修复失败后升级 Architect/人工

## 默认闸门套装

### 前端项目 (React/Vue/Angular)

```yaml
gates:
  - id: lint
    name: ESLint 检查
    command: npm run lint
    autofix: npm run lint:fix
    fail_action: auto_fix  # 自动修复优先
    auto_retry: 2
    escalation: manual     # 自动修复失败后转人工
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"lint","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl
    
  - id: format
    name: Prettier 格式化
    command: npm run format:check
    autofix: npm run format
    fail_action: auto_fix
    auto_retry: 2
    escalation: manual
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"format","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl
    
  - id: type-check
    name: TypeScript 类型检查
    command: npx tsc --noEmit
    autofix: null
    fail_action: block     # 类型错误必须人工修复
    auto_retry: 0
    escalation: architect  # 升级给架构师
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"type-check","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl
    
  - id: unit-test
    name: 单元测试
    command: npm run test:unit
    autofix: null
    fail_action: block
    auto_retry: 2          # Agent 最多尝试 2 次修复
    escalation: architect  # 2 次失败后升级
    fail_classification:    # 失败分类逻辑
      - pattern: "AssertionError|expect|assert"  # 测试代码错误
        handler: agent_fix                              # Agent 自修
      - pattern: ".*"                                   # 其他（业务逻辑错误）
        handler: agent_fix_with_limit                   # Agent 修，有限次
    threshold:
      coverage: 80
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"unit-test","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl
      # 同时记录 fix_attempt（在修复逻辑中调用）
    
  - id: build
    name: 生产构建
    command: npm run build
    autofix: null
    fail_action: block
    auto_retry: 0
    escalation: architect
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"build","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl
```

### 后端项目 (Node.js/Python/Go)

```yaml
gates:
  - id: lint
    name: 代码规范
    command: "[npm run lint / make lint / golangci-lint run]"
    autofix: "[npm run lint:fix]"
    fail_action: auto_fix
    auto_retry: 2
    escalation: manual
    
  - id: unit-test
    name: 单元测试
    command: "[npm test / pytest / go test ./...]"
    autofix: null
    fail_action: block
    auto_retry: 2
    escalation: architect
    fail_classification:
      - pattern: "assert|AssertionError|mock"  # 测试代码错误
        handler: agent_fix
      - pattern: ".*"
        handler: agent_fix_with_limit
    threshold:
      coverage: 70
      
  - id: integration-test
    name: 集成测试
    command: "[npm run test:integration]"
    autofix: null
    fail_action: warn      # 集成测试失败不阻塞
    auto_retry: 1
    escalation: record     # 记录待修复项
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"integration-test","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl
    
  - id: type-check
    name: 类型检查
    command: "[tsc --noEmit / mypy / go vet ./...]"
    autofix: null
    fail_action: block
    auto_retry: 0
    escalation: architect
```

### 全栈项目

全栈项目使用前后端闸门的并集，分阶段运行：

```yaml
phases:
  backend:
    gates: [lint, type-check, unit-test]
  frontend:
    gates: [lint, format, type-check, unit-test, build]
  integration:
    gates: [integration-test, e2e-test]
```

## 失败处理流程

### 自动修复类（lint/format）

```
失败 → 调用 autofix → 重试（第 1 次）
                ↓
           成功? → 是: 标记 pass
                ↓ 否
           重试（第 2 次）
                ↓
           成功? → 是: 标记 pass
                ↓ 否
           标记 blocked
           记录到 state.md blockers
           转人工处理
```

### 测试类（unit-test）

```
失败 → 分类失败类型：

  A) 测试代码错误（断言/模拟/测试逻辑）
     → Agent 自修测试代码
     → 重试运行
     → 通过? → 是: done / 否: 回到分类

  B) 业务逻辑错误
     → Agent 分析根因
     → 尝试修复（第 1 次）
     → 重试运行
            ↓
       通过? → 是: done
            ↓ 否
       尝试修复（第 2 次）
       → 重试运行
            ↓
       通过? → 是: done
            ↓ 否
       标记 blocked
       @Architect 或转人工
       记录到 state.md blockers（含已尝试的方案）
```

### 警告类（integration-test）

```
失败 → 记录警告
   → 不阻塞流程
   → 写入 state.md 的 warn 区域
   → 必须在后续 sprint 中修复
```

## 自定义闸门

在 `.cc-loop/gates.yml` 中添加自定义检查：

```yaml
gates:
  - id: security-scan
    name: 安全扫描
    command: npm audit --audit-level=moderate
    autofix: null
    fail_action: block
    auto_retry: 0
    escalation: manual
    
  - id: doc-check
    name: 文档完整性
    command: test -f README.md && test -f API.md
    autofix: null
    fail_action: warn
    auto_retry: 0
    escalation: record
    
  - id: performance
    name: 性能基准
    command: npm run benchmark
    autofix: null
    fail_action: warn
    auto_retry: 0
    escalation: record
    threshold:
      maxResponseTime: 100  # ms
```

## 安全专项闸门（SEC 专属）

当项目激活 SEC（安全工程师）扩展角色时，引入以下安全闸门：

### 基础安全闸门（推荐必配）

```yaml
gates:
  - id: security-scan
    name: 依赖安全扫描
    command: npm audit --audit-level=moderate
    autofix: npm audit fix
    fail_action: block
    auto_retry: 1
    escalation: sec           # 升级给 SEC 分析
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"security-scan","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl

  - id: secret-check
    name: 密钥硬编码检查
    command: git-secrets --scan || truffleHog filesystem . --no-verification
    autofix: null
    fail_action: block
    auto_retry: 0
    escalation: sec
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"secret-check","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl

  - id: sast
    name: 静态应用安全测试
    command: semgrep --config=auto . || bandit -r . || gosec ./...
    autofix: null
    fail_action: block
    auto_retry: 0
    escalation: sec
    fail_classification:
      - pattern: "high|critical|severity: HIGH"
        handler: block              # 高危必须修复
      - pattern: "medium|severity: MEDIUM"
        handler: warn               # 中危记录 warn
      - pattern: "low|severity: LOW"
        handler: record             # 低危记录待修复
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"sast","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl

  - id: compliance-check
    name: 合规基线检查
    command: scripts/compliance-check.sh
    autofix: null
    fail_action: block
    auto_retry: 0
    escalation: sec
    metrics_cmd: |
      echo '{"ts":"'$(date -u +%Y-%m-%dT%H:%M:%SZ)'","type":"gate","gate":"compliance-check","task_id":"'${TASK_ID}'","status":"'${STATUS}'","round":'${ROUND}',"agent":"'${AGENT}'","duration_ms":'${DURATION:-0}'}' >> .cc-loop/metrics.jsonl
```

### 安全闸门失败处理

```
security-scan 失败（依赖漏洞）：
  → 尝试 autofix（npm audit fix）
  → 重试运行
  → 仍失败 → SEC 分析漏洞影响
    → 可升级修复 → SEC 提供修复方案 → Developer 执行
    → 无修复版本 → SEC 评估风险 → PM 决策是否接受并记录

secret-check 失败（密钥硬编码）：
  → 立即标记 blocked
  → 不可自动修复
  → SEC 定位泄露位置 → Developer 迁移到密钥管理服务
  → 如已提交历史，评估是否需要轮换密钥

sast 失败（静态扫描）：
  → 按严重度分类：
    high/critical → 必须修复，标记 blocked
    medium → 记录 warn，本轮不阻塞，但必须在下轮前修复
    low → 记录待修复清单，不阻塞
  → SEC 提供修复指导

compliance-check 失败（合规基线）：
  → 立即标记 blocked
  → SEC 输出不合规项清单
  → BA 确认业务影响（如涉及合规义务）
  → Architect 设计合规方案
  → Developer 实现
  → SEC 复验
```

### 安全闸门与扩展角色的关系

| 安全闸门 | 触发条件 | 执行角色 | 升级路径 |
|---------|---------|---------|---------|
| security-scan | 每次构建 | SEC 主导 | SEC 分析 → Developer 修复 |
| secret-check | 每次提交 | SEC 主导 | 立即 blocked → Developer 修复 |
| sast | 每次 PR | SEC 主导 | 按严重度分级处理 |
| compliance-check | 阶段切换/上线前 | SEC 主导 | SEC + BA 联合评估 → Architect 方案 |

---

## 闸门状态语义

| 状态 | 含义 | 下一步动作 |
|------|------|-----------|
| pending | 尚未执行检查 | 等待执行 |
| running | 检查进行中 | 等待完成 |
| pass | 检查通过 | 继续流程 |
| auto_fixing | 自动修复中 | 等待修复完成重试 |
| fail-block | 检查失败，阻塞 | 进入修复循环或升级 |
| fail-warn | 检查失败，警告 | 记录并继续 |
| blocked | 已升级，等待人工 | 等待 Architect/人工决策 |
| skipped | 跳过（人工豁免） | 继续流程，记录豁免原因 |

## 在 /loop 中集成闸门

```
/loop

引用：.cc-loop/goal.md, .cc-loop/SUMMARY

每轮执行：
1. 完成当前开发任务
2. Coordinator 读取 .cc-loop/gates.yml
3. 按阶段运行对应闸门检查：
   - lint/format: 自动修复模式（max 2 次重试）
   - unit-test: 分类处理（测试错误自修 / 逻辑错误 max 2 次后升级）
   - integration: 记录 warn，不阻塞
4. Coordinator 更新 state.md 中的闸门状态表
5. 如有 fail-block：
   - 将修复任务加入当前轮次目标
   - 指定修复 Agent
   - 继续循环修复
6. 如无失败：标记任务完成，选择下一任务
```

## 升级路径

当闸门失败且自动处理达到上限时：

```
Agent 修复失败（2 次）
  → Coordinator 标记 blocked
  → 写入 state.md blockers（含失败原因、已尝试方案）
  → 根据 escalation 配置：
    - architect: @Architect 分析并给出方案
    - manual: 转人工决策
    - record: 记录到待修复清单，不阻塞当前流程
  → 解决后重置闸门状态为 pending，重新运行
```
