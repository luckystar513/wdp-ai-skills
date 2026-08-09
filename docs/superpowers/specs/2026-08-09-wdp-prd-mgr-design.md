# wdp-prd-mgr 需求管理 Skill 设计（MVP）

- 日期：2026-08-09
- 状态：MVP 设计
- 目标目录：`wdp-prd-mgr/`

## 1. 背景与目标

用户需要一套基于 superpowers 的需求管理能力，核心痛点：

> 每次只能等着当前迭代完成，再进入下一轮需求讨论。

目标：

1. **持续讨论需求**：随时 brainstorm 新需求，产出的设计先存起来，不被当前实现阻塞。
2. **存储为长期资产**：需求卡片放**用户全局位置**（重装/迁移不丢失），但**按项目分目录**（不放在项目目录下）。
3. **并行实现**：确认的需求通过**后台子代理**实现（worktree 隔离），主会话可继续讨论。
4. **依赖与冲突感知**：派发前自动计算调度状态——哪些可立即并行、哪些阻塞等谁、哪些与执行中任务冲突。
5. **受控验收**：集成测试 + code review + 用户确认，防止子代理变更失控。

## 2. 定位：与 superpowers 的关系

- superpowers 是**任务执行器**：一次一个功能，brainstorm → spec → plan → execute → verify → 收尾。
- wdp-prd-mgr 是**调度层**：管理一堆需求的 backlog、依赖、并发、顺序，**叠加在 superpowers 之上**，不重造轮子。
- 复用 superpowers 已有 skill：`brainstorming`、`writing-plans`、`test-driven-development`、`requesting-code-review`、`verification-before-completion`。
- 实现方式是**纯指令型 skill**（单个 `SKILL.md`），指示 Claude 在需要时调用上述组件。

## 3. 存储模型

### 3.1 全局根 + 项目子目录

```
C:\Users\admin\.wdp\prd\                 ← 全局根（可用环境变量 PRD_ROOT 覆盖）
  <project-slug>\                        ← 每个项目一个子目录
    backlog\
      R-001.md ... R-NNN.md              ← 需求卡片，每需求一个文件
    execution-plan.md                    ← 派生的调度视图（每次变更后重写）
    plans\                               ← 派发时子代理生成的实现计划存档
```

### 3.2 项目指针文件（桥接机制）

每个项目根目录放一个指针文件 `.wdp-prd`，内容为该项目的全局 backlog 绝对路径：

```
<项目>\.wdp-prd  → 内容: C:\Users\admin\.wdp\prd\cc-understand
```

skill 每次调用流程：读 `<cwd>/.wdp-prd` → 拿到本项目 backlog 路径 → 操作它。
指针文件不存在 → 首次使用初始化：问项目名 → 创建全局目录 → 写指针文件。

### 3.3 存储位置优先级

1. 项目内 `.wdp-prd` 指针文件（绝对路径）
2. 若指针文件缺失 → 初始化流程创建

## 4. 需求卡片数据模型

### 4.1 卡片 frontmatter

```markdown
---
id: R-001
title: 登录失败重试机制
status: draft
priority: high            # high / medium / low
depends_on: [R-002]       # 依赖的需求 id；调度的输入
conflicts_with: []        # 语义冲突的需求 id；调度的输入
created: 2026-08-09       # 创建时写一次
updated: 2026-08-09       # 每次改动卡片时顺手更新
---
```

### 4.2 卡片正文

```markdown
## 背景 / 目标        ← brainstorming 讨论产物
## 验收标准          ← 可测试的验收条件
## 设计文档          → docs/superpowers/specs/<date>-<topic>-design.md（外链）
## 实现计划          → prd/plans/<id>.md（派发时生成）
## 验收记录          → 合并后填：测试结果 / review 结论
```

## 5. 状态机

```
draft ──/wdp-prd-confirm──▶ confirmed ──/wdp-prd-dispatch──▶ in-progress
  │                           │                                   │
  └─ cancelled                └─ blocked（依赖未满足）◀──失败──────┘
                                                     │
in-progress ──子代理报告──▶ needs-review ──/wdp-prd-accept──▶ accepted ──▶ done
```

- `draft`：刚讨论完，未确认
- `confirmed`：已确认，可派发
- `in-progress`：后台子代理开发中
- `needs-review`：子代理报告回来，待验收
- `accepted`：验收通过
- `done`：已合入/完成
- `blocked`：依赖未满足或子代理失败
- `cancelled`：取消

## 6. 命令集（全部带 wdp- 前缀）

| 命令 | 作用 |
|------|------|
| `/wdp-prd` | 主入口：状态总览 + 快捷命令提示 |
| `/wdp-prd-new` | 调用 brainstorming 讨论新需求 → 存卡片（draft） |
| `/wdp-prd-list` | backlog 摘要（id/title/status/priority/依赖） |
| `/wdp-prd-confirm <id>` | 确认需求 → confirmed |
| `/wdp-prd-dispatch` | 出调度状态报告 → 用户挑选 → 后台子代理派发 |
| `/wdp-prd-accept <id>` | 验收：集成测试 + code review + 用户确认 |
| `/wdp-prd-update <id>` | 修改卡片（依赖/优先级/状态） |

## 7. 调度算法（MVP 只做两条规则）

每次 `dispatch` 或任何卡片变更后，Claude 读全部卡片，按两条规则计算，**不做拓扑排序分波次**：

1. `depends_on` 中仍有**非 done 状态**的任务 → 报 `⏳ 等 R-00X 完成`
   （注：被 `cancelled` 的依赖视为**已满足**，不再阻塞。）
2. `conflicts_with` 中有 **in-progress** 任务 → 报 `⚠ 与 R-00X(开发中) 冲突，需等它完成或用户裁定`

**可立即派发** = `status==confirmed` + 依赖全满足 + 与 in-progress 无冲突 + 并发数未超上限。
**同批约束**：用户挑选的一批必须**两两无冲突**；若选了互相 `conflicts_with` 的一对，提示并强制只选一个。
**并发上限**：默认 2，可配置。超出报"已达并发上限，先验收手头的"。

**调度状态报告格式**：

```
▶ 现在就能并行派发      [R-002, R-003]
⏳ 依赖阻塞，等任务完成  R-005 → 等 R-002(开发中)
⚠ 与执行中任务冲突      R-007 ↔ R-008(开发中)
```

## 8. 派发流程

1. 出调度状态报告（见 §7）给用户看。
2. 用户挑选一批"立即派发"的需求。
3. 每需求 spawn 一个**后台子代理**，在 **git worktree 隔离分支**里独立执行：
   `writing-plans → test-driven-development → 开发 → verification-before-completion`。
4. 子代理返回报告（改了什么、跑了哪些测试、结果如何）。
5. 卡片标记 `in-progress`，重写 `execution-plan.md`。

## 9. 验收流程

1. 子代理报告回来 → 卡片标 `needs-review`。
2. 主会话合并 worktree 分支 / 应用变更。
3. 跑**全量集成测试**。
4. 走 **code review**（复用 `requesting-code-review`）。
5. 呈现变更摘要给用户 → 用户最终确认。
6. 确认后标 `accepted → done`，重写 `execution-plan.md`，记录验收记录。

## 10. 错误处理

| 情况 | 处理 |
|------|------|
| 非 git 仓库 | 无法 worktree 隔离 → 明确警告，降级为串行/非隔离派发 |
| 子代理失败 | 卡片标 `blocked` + 报告失败原因，更新 execution-plan |
| 依赖成环 | 检测并报告，不派发 |
| 冲突需求同时确认 | 派发时阻塞并请用户裁定 |
| 无 Agent 工具 | 降级为手动引导流程 |
| `.wdp-prd` 指针缺失 | 触发初始化流程 |

## 11. 测试

skill 的测试 = **场景化演练**：在临时目录造一批模拟卡片，跑 `list / confirm / dispatch / accept` 全流程，验证：
- 状态机迁移正确
- 调度状态报告正确（可并行/阻塞/冲突判定）
- 错误分支（依赖成环、非 git、子代理失败）
- 验收流程闸门生效

演练脚本记录在 `references/test-scenarios.md`。

## 12. Skill 自身文件结构

```
wdp-prd-mgr/
  SKILL.md                          ← 主指令（frontmatter: name, description）
  references/
    card-template.md                ← 卡片模板
    test-scenarios.md               ← 演练场景
  scripts/                          ← 预留（当前为空，v2 放调度脚本）
```

## 13. 后续迭代清单（本次砍掉，升级参考）

1. **自动拓扑排序 + 波次分组**：依赖多时自动算全序，分组并行。
2. **冲突自动裁定流程**：冲突需求自动串行 + 用户裁定界面化。
3. **调度脚本化**：Python/Node 脚本做拓扑排序（backlog >100 条时）。
4. **确认后自动预生成计划**：后台子代理提前写计划并存档，派发时直接用。
5. **拆分为多个子 skill**：`/wdp-prd-brainstorm`、`/wdp-prd-schedule` 等。
6. **对接外部 PM 工具**：GitHub Issues / Jira 双向同步。
7. **卡片内嵌状态变更日志**：非 git 场景的审计需求。

## 14. 已知决策记录

| 决策点 | 结论 |
|-------|------|
| 并发模型 | 显式派发 + 后台并行子代理（worktree 隔离），主会话继续 brainstorm |
| 存储模型 | 全局根 + 项目子目录 + `.wdp-prd` 指针 |
| 触发模型 | 派发时出调度报告 → 用户挑 → 后台执行 |
| 执行 | 全部走子代理 |
| 验收 | 集成测试 + code review + 用户确认 |
| 时间戳 | `created` + `updated`；状态变更历史记入迭代清单 |
| 命名 | 所有命令带 `wdp-` 前缀 |
| 定位 | superpowers 之上的调度层，纯指令单 skill，复用已有组件 |
