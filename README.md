# cc-loop

> 通过状态文件驱动、质量闸门控制、多 Agent 协作，将 Claude Code 的 `/loop` 和 `/go` 转化为工业级项目开发流程。

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Claude Code Skill](https://img.shields.io/badge/Claude-Code%20Skill-blueviolet)](https://docs.claude.com/en/docs/claude-code/skills)

## 这是什么？

`cc-loop` 是一个 **Claude Code Skill**，用于管理长时间、多轮迭代的复杂项目开发。它将 `/loop` 和 `/go` 这两个原生命令，结合**状态文件驱动**、**质量闸门控制**和**多 Agent 角色协作**，包装成可控、可追踪、高质量的工业级工作流。

### 核心特性

- 📋 **状态文件驱动** — `.cc-loop/state.md` 作为单点真相源（Single Source of Truth），每轮自动更新
- 🚦 **质量闸门** — 预定义通过标准（lint / test / coverage），未通过强制修复
- 👥 **多 Agent 协作** —「7 + N」弹性角色团队（架构师 / 开发者 / 测试者 / 审查者 / 协调者 / 产品 / 运维 + 扩展角色）
- 🔁 **循环与单点模式** — `/loop` 用于持续迭代，`/go` 用于 1-3 轮单点突破
- 🛡️ **Change Plan 守门** — 默认保守模式，修改代码前必须输出计划并获得用户确认
- 🚨 **死循环检测** — Coordinator 自动检测 3 轮无推进，触发升级

## 适用场景

| 场景 | 推荐模式 |
|------|---------|
| 长时间功能开发、多轮迭代 | `/loop` 或 `/loop --parallel` |
| 多 Agent 并行冲刺 | `/loop --parallel` |
| Bug 修复（≤3 轮） | `/go` |
| 单点技术攻坚、架构决策 | `/go Architect` |
| 紧急热修复 | `/go` |

## 快速开始

### 1. 安装

将 `cc-loop/` 目录复制到你的项目根目录（或者通过 Claude Code skills 机制加载）：

```bash
# 方式 A：作为项目级 skill
cp -r cc-loop /path/to/your/project/.claude/skills/cc-loop

# 方式 B：作为个人 skill
cp -r cc-loop ~/.claude/skills/cc-loop
```

### 2. 初始化项目

进入你的项目目录，首次使用前必须完成**启动检查单**：

```bash
mkdir -p .cc-loop
# 创建 goal.md（定义项目目标和验收标准）
# 创建 state.md（使用 references/project-state-schema.md 模板）
# 创建 gates.yml（使用 references/quality-gates.md 默认配置）
```

### 3. 启动循环

```
/loop

引用：.cc-loop/goal.md, .cc-loop/SUMMARY

启动协议：
1. Coordinator 读取 SUMMARY，确定当前轮次和 Agent 分配
2. 活跃 Agent 先输出 Change Plan → [用户确认] → 执行
3. Coordinator 更新 state.md，运行质量闸门
4. 全部通过 → 推进 / 有失败 → 修复循环
```

详细使用方法参见 [SKILL.md](./SKILL.md)。

## 目录结构

```
cc-loop/
├── SKILL.md                          # 技能主文件（Claude Code 加载入口）
├── LICENSE                           # MIT 协议
├── README.md                         # 本文件
└── references/                       # 详细参考文档
    ├── project-state-schema.md       # state.md 完整格式规范
    ├── state-management.md           # 三级读取、Summary 协议
    ├── quality-gates.md              # 闸门配置、自动修复、升级逻辑
    ├── agent-roles.md                # 「7 + N」角色团队完整定义
    ├── prompt-templates.md           # 核心+扩展角色系统提示模板
    ├── loop-patterns.md              # 7 种常用循环模式模板
    ├── troubleshooting.md            # 4 个逃生舱、故障恢复流程
    └── metrics.md                    # 度量指标定义、采集规范
```

## 核心概念

| 概念 | 说明 |
|------|------|
| **状态文件** | 单点真相源，记录项目背景、当前阶段、任务列表、质量检查结果 |
| **质量闸门** | 预定义的通过标准，每轮迭代必须通过才能进入下一阶段 |
| **Agent 角色** | 「7 + N」弹性团队（7 核心 + N 扩展），通过系统提示分配角色 |
| **循环模式** | `/loop` 用于多步骤持续迭代，`/go` 用于单任务自主执行 |
| **Change Plan** | 变更计划，Agent 修改代码前必须输出计划，用户确认后才执行 |
| **Coordinator** | 协调者 Agent，专责状态文件维护、摘要更新、轮次管理 |

## 任务状态机

```
backlog → claimed → in-progress → ready-for-review → reviewing
                                              ↓           ↓
                                          ready-for-test  [block] 退回
                                              ↓
                                          testing → done
```

## 文档导航

- [SKILL.md](./SKILL.md) — 技能主文档（必读）
- [references/agent-roles.md](./references/agent-roles.md) — 角色团队定义
- [references/quality-gates.md](./references/quality-gates.md) — 闸门配置
- [references/loop-patterns.md](./references/loop-patterns.md) — 7 种循环模式
- [references/troubleshooting.md](./references/troubleshooting.md) — 故障排除

## 贡献

欢迎贡献！请通过 Issue 提交问题或 PR 提交改进。

## 许可

本项目采用 [MIT License](./LICENSE) 开源。
