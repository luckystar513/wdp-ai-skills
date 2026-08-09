# wdp-prd-mgr Skill 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 交付一个纯指令型需求管理 skill（`SKILL.md` + `references/`），让用户持续 brainstorm 积累需求卡片，按依赖/冲突自动算调度状态，确认后派发给后台子代理并行实现，集成测试 + review + 确认验收。

**Architecture:** 单 `SKILL.md` 编排，指示 Claude 复用 superpowers 已有组件（brainstorming / writing-plans / test-driven-development / requesting-code-review / verification-before-completion）。需求卡片存用户全局（`~/.wdp/prd/<project-slug>/`），项目内 `.wdp-prd` 指针文件桥接。MVP 调度只做两条规则 + 并发上限，不做拓扑排序。

**Tech Stack:** 无代码。交付物全部是 Markdown 指令文件。测试 = 场景化干跑（scratch 项目 + git）。

## Global Constraints

- 所有命令带 `wdp-` 前缀：`/wdp-prd`、`/wdp-prd-new`、`/wdp-prd-list`、`/wdp-prd-confirm <id>`、`/wdp-prd-dispatch`、`/wdp-prd-accept <id>`、`/wdp-prd-update <id>`。
- 默认全局根 `~/.wdp/prd/`，可用环境变量 `PRD_ROOT` 覆盖；每项目一个子目录（slug）；项目根 `.wdp-prd` 指针文件存该项目的全局绝对路径。
- 卡片 frontmatter 字段（精确）：`id`、`title`、`status`、`priority`(high/medium/low)、`depends_on`、`conflicts_with`、`created`、`updated`。`created` 创建时写一次；`updated` 每次改动更新。
- 状态机（精确值）：`draft`、`confirmed`、`in-progress`、`needs-review`、`accepted`、`done`，外加 `blocked`、`cancelled`。
- 调度两条规则：① `depends_on` 有非 done → 阻塞（`cancelled` 依赖视为已满足）；② `conflicts_with` 有 in-progress → 冲突。可派发 = confirmed + 依赖满足 + 与 in-progress 无冲突 + 并发 < 上限。并发上限默认 2。同批必须两两无冲突。
- 执行全部走后台子代理（git worktree 隔离）；验收 = 集成测试 + code review + 用户确认。
- SKILL.md 正文用中文；frontmatter `description` 中英双语（供 skill 发现）。
- 不在本项目实现任何拓扑排序/冲突自动裁定/脚本（记录于 spec §13 后续迭代清单）。

---

### Task 1: 骨架 + 存储模型 + 卡片模板 + 初始化

**Files:**
- Create: `SKILL.md`
- Create: `references/card-template.md`
- Create: `references/test-scenarios.md`

**Interfaces:**
- Produces: `SKILL.md` 的 frontmatter（name: `wdp-prd-mgr`）、存储模型、初始化流程；`references/card-template.md` 卡片模板；`references/test-scenarios.md` 场景 S1。

- [ ] **Step 1: 写场景 S1（初始化 + 建卡）到 `references/test-scenarios.md`**

```markdown
# wdp-prd-mgr 测试场景

所有场景在临时 scratch 项目目录干跑。每个场景：Setup（准备环境）→ Action（按 SKILL.md 对应命令段落执行）→ Expected（检查不变量）。

## S1 初始化 + 建卡
- Setup: 空 scratch 项目 `dryrun-1/`（无 `.wdp-prd` 指针）。
- Action: 按 SKILL.md「初始化」执行；再按「/wdp-prd-new」创建一个需求。
- Expected:
  1. 创建 `<PRD_ROOT>/<slug>/` 含 `backlog/`、`plans/`、`execution-plan.md`。
  2. `.wdp-prd` 内容 = 该全局绝对路径。
  3. `backlog/R-001.md` 存在，frontmatter 字段齐全（id/title/status/priority/depends_on/conflicts_with/created/updated），status=draft。
  4. created 与 updated 均为当天日期。
```

- [ ] **Step 2: 干跑 S1 → 确认失败（SKILL.md 尚不存在，无可执行指令）**

- [ ] **Step 3: 写 `SKILL.md` 骨架 + 存储模型 + 初始化**

```markdown
---
name: wdp-prd-mgr
description: 需求管理 backlog（基于 superpowers）。持续 brainstorm 积累需求卡片，按依赖/冲突算调度状态，确认后派发给后台子代理并行实现。Use when you want to keep discussing new requirements while confirmed ones are implemented in parallel via subagents.
---

# wdp-prd-mgr — 需求管理

基于 superpowers 的需求管理：**讨论与实现解耦**。随时 brainstorm 新需求存为卡片；确认后通过后台子代理并行实现，主会话不被阻塞。

## 何时使用
- 讨论/记录新需求（不立即实现）→ `/wdp-prd-new`
- 查看 backlog / 调度状态 → `/wdp-prd` 或 `/wdp-prd-list`
- 确认需求可派发 → `/wdp-prd-confirm <id>`
- 派发实现 → `/wdp-prd-dispatch`
- 验收完成的需求 → `/wdp-prd-accept <id>`
- 修改卡片 → `/wdp-prd-update <id>`

## 存储模型
- 全局根：`~/.wdp/prd/`（可用环境变量 `PRD_ROOT` 覆盖）。
- 项目子目录：`<PRD_ROOT>/<project-slug>/`，内含：
  - `backlog/R-###.md` 需求卡片
  - `execution-plan.md` 调度视图
  - `plans/` 实现计划存档
- 桥接：项目根 `.wdp-prd` 文件，内容为该项目的全局 backlog 绝对路径。每次命令操作前先读它。

### 初始化
若 `<cwd>/.wdp-prd` 不存在：
1. 询问项目名（slug，小写连字符）。
2. 创建 `<PRD_ROOT>/<slug>/`（含 `backlog/`、`plans/`）与空 `execution-plan.md`。
3. 写 `.wdp-prd`，内容为全局绝对路径。
4. 报告初始化完成。

## 状态机
`draft → confirmed → in-progress → needs-review → accepted → done`
另有 `blocked`（依赖未满足 / 子代理失败）、`cancelled`。

## 需求卡片
格式见 `references/card-template.md`。创建时复制模板。

### /wdp-prd-new —— 讨论并新建需求
1. 确认本项目的 `.wdp-prd` 指针存在（缺失则先走「初始化」）。
2. 调用 `superpowers:brainstorming` 与用户讨论需求，产出设计文档（存到项目 `docs/superpowers/specs/`）。
3. 按 `references/card-template.md` 创建新卡片 `backlog/R-###.md`：
   - id 自增（当前最大 +1）；status=draft；priority 由用户定或默认 medium。
   - created/updated = 当天。
   - 「设计文档」段写入 spec 路径。
4. 重写 `execution-plan.md`。
```

- [ ] **Step 4: 写 `references/card-template.md`**

```markdown
# 需求卡片模板

复制为 `backlog/R-###.md`。id 自增，从 R-001 开始，不回收。

```markdown
---
id: R-001
title: <一句话描述需求>
status: draft
priority: medium              # high / medium / low
depends_on: []                # 依赖的需求 id，如 [R-002]
conflicts_with: []            # 语义冲突的需求 id
created: YYYY-MM-DD           # 创建时写一次
updated: YYYY-MM-DD           # 每次改动更新
---

## 背景 / 目标
## 验收标准
## 设计文档          → 外链 docs/superpowers/specs/<date>-<topic>-design.md
## 实现计划          → prd/plans/<id>.md（派发时生成）
## 验收记录          → 合并后填：测试结果 / review 结论
```
```

- [ ] **Step 5: 重跑 S1 → 通过**

- [ ] **Step 6: Commit**

```bash
git add SKILL.md references/card-template.md references/test-scenarios.md
git commit -m "feat: skill 骨架、存储模型、卡片模板、初始化"
```

---

### Task 2: 总览 + 列表 + 更新命令

**Files:**
- Modify: `SKILL.md`（在「状态机」后新增命令段落）
- Modify: `references/test-scenarios.md`

**Interfaces:**
- Consumes: Task 1 的卡片模板（frontmatter 字段）、存储模型。
- Produces: `/wdp-prd`、`/wdp-prd-list`、`/wdp-prd-update` 三段指令；场景 S2。

- [ ] **Step 1: 写场景 S2（列表 + 更新）到 test-scenarios.md**

```markdown
## S2 列表 + 更新
- Setup: 已有 3 张卡片 R-001(draft,high)、R-002(draft,low)、R-003(confirmed)。
- Action: 执行 `/wdp-prd-list`；再执行 `/wdp-prd-update R-002`（priority 改 high，status 改 confirmed）。
- Expected:
  1. list 输出表格含 id/title/status/priority/依赖，3 行齐全、字段正确。
  2. update 后 R-002 的 priority=high、status=confirmed，updated=当天。
  3. created 未被改动。
- Action: 执行 `/wdp-prd`（主入口）。
- Expected: 显示 backlog 摘要 + 各命令提示。
```

- [ ] **Step 2: 干跑 S2 → 失败**

- [ ] **Step 3: 在 SKILL.md 增加三个命令段落**

```markdown
## 命令

### /wdp-prd —— 主入口
1. 读所有卡片，输出 backlog 摘要表格（id/title/status/priority/依赖）。
2. 输出当前调度状态报告（规则见「调度规则」）。
3. 提示可用的 `/wdp-prd-*` 命令。

### /wdp-prd-list —— 列表
输出 backlog 摘要表格，按 priority 降序、created 升序排序。若卡片为空，明确说明。

### /wdp-prd-update <id> —— 修改卡片
1. 定位卡片（不存在则报错并列出最近 id）。
2. 询问要改的字段（status/priority/depends_on/conflicts_with/title）。
3. 校验取值：status 必须属于状态机集合；priority 必须 high/medium/low。
4. 更新字段，`updated` 置为当天，`created` 保持不变。
5. 重写 `execution-plan.md`（如受影响）。
```

- [ ] **Step 4: 重跑 S2 → 通过**

- [ ] **Step 5: Commit**

```bash
git add SKILL.md references/test-scenarios.md
git commit -m "feat: 主入口、列表、更新命令"
```

---

### Task 3: 确认命令 + 调度规则

**Files:**
- Modify: `SKILL.md`（新增「调度规则」段 + `/wdp-prd-confirm`）
- Modify: `references/test-scenarios.md`

**Interfaces:**
- Consumes: 状态机、卡片依赖字段。
- Produces: 「调度规则」段落（Task 4/5 的 dispatch/accept 复用）；场景 S3。

- [ ] **Step 1: 写场景 S3（调度报告：依赖/冲突/cancelled 依赖）**

```markdown
## S3 调度报告
- Setup:
  - R-001 confirmed，无依赖。
  - R-002 confirmed，无依赖，且 R-001.conflicts_with=[R-002]。
  - R-005 confirmed，depends_on=[R-002]，R-002 状态 in-progress。
  - R-007 confirmed，depends_on=[R-099]，R-099 状态 cancelled。
- Action: 执行 `/wdp-prd-dispatch`（仅输出报告，不派发）。
- Expected:
  1. 可立即派发 = [R-001, R-002]（互为冲突的一对都列出，但提示「二者冲突，只能挑一个」）。
  2. R-005 → 报「⏳ 等 R-002(开发中) 完成」。
  3. R-007 → 不阻塞（cancelled 依赖视为已满足），列入可派发。
```

- [ ] **Step 2: 干跑 S3 → 失败**

- [ ] **Step 3: 在 SKILL.md 增加「调度规则」段 + `/wdp-prd-confirm`**

```markdown
## 调度规则
对全部卡片计算状态报告，两条规则（MVP 不做拓扑排序）：
1. `depends_on` 中仍有非 done 状态 → `⏳ 等 R-00X 完成`（被 `cancelled` 的依赖视为已满足，不阻塞）。
2. `conflicts_with` 中有 in-progress → `⚠ 与 R-00X(开发中) 冲突，需等它完成或用户裁定`。

**可立即派发** = status==confirmed + 依赖全满足 + 与 in-progress 无冲突 + in-progress 数量 < 并发上限。
**并发上限** 默认 2，可配置（环境变量 `PRD_MAX_CONCURRENT`）。超出报「已达并发上限，先验收手头的」。
**同批约束**：用户挑选的一批必须两两无冲突；选了互相 conflicts_with 的一对时，强制只挑一个。

报告格式：
```
▶ 现在就能并行派发      [R-001, R-002]
⏳ 依赖阻塞，等任务完成  R-005 → 等 R-002(开发中)
⚠ 与执行中任务冲突      R-003 ↔ R-004(开发中)
```

### /wdp-prd-confirm <id> —— 确认需求
1. 卡片存在且 status==draft。
2. 确认后 status→confirmed，`updated` 更新。
3. 重写 `execution-plan.md`。
```

- [ ] **Step 4: 重跑 S3 → 通过**

- [ ] **Step 5: Commit**

```bash
git add SKILL.md references/test-scenarios.md
git commit -m "feat: 确认命令与调度规则"
```

---

### Task 4: 派发流程

**Files:**
- Modify: `SKILL.md`（新增 `/wdp-prd-dispatch` 完整流程 + execution-plan.md 重写规则）
- Modify: `references/test-scenarios.md`

**Interfaces:**
- Consumes: 调度规则（Task 3）。
- Produces: 派发流程 + execution-plan.md 格式；场景 S4。

- [ ] **Step 1: 写场景 S4（派发）**

```markdown
## S4 派发
- Setup: scratch 项目 `dryrun-4/`，git 仓库；R-001、R-002 均 confirmed 无冲突无依赖。
- Action: 执行 `/wdp-prd-dispatch`，挑选 [R-001, R-002]。
- Expected:
  1. 报告正确展示可派发列表；并发上限未超。
  2. 各派发一个后台子代理；R-001/R-002 状态 → in-progress。
  3. `execution-plan.md` 重写：含 in-progress=[R-001,R-002]、可派发=[]、阻塞=[]、冲突=[]。
  4. 子代理返回报告后卡片 → needs-review。
- Action: 非 git 项目重跑 dispatch（或 git 仓库但删除隔离能力）。
- Expected: 明确警告「无法 worktree 隔离，降级为串行/非隔离」，不静默失败。
```

- [ ] **Step 2: 干跑 S4 → 失败**

- [ ] **Step 3: 在 SKILL.md 增加 `/wdp-prd-dispatch` 与 execution-plan 规则**

```markdown
## execution-plan.md
每次状态变更/派发/验收后由 Claude 重写：
```markdown
# 执行计划（自动生成，勿手改）
更新时间: YYYY-MM-DD
in-progress: [R-001, R-002]
可立即派发: []
阻塞: R-005 → 等 R-002
冲突: []
```

### /wdp-prd-dispatch —— 派发
1. 出调度状态报告（规则见「调度规则」）。
2. 用户挑选一批「立即派发」（同批两两无冲突；并发超上限时提示先验收）。
3. 每需求 spawn 一个**后台子代理**，在 **git worktree 隔离分支**中执行：
   `superpowers:writing-plans` → `superpowers:test-driven-development` → 开发 → `superpowers:verification-before-completion`。
4. 子代理返回报告：改了什么文件、跑了哪些测试、结果、遗留问题。
5. 卡片 → in-progress，`updated` 更新，重写 execution-plan.md。
6. 子代理完成后：卡片 → needs-review，提示用户 `/wdp-prd-accept <id>`。

**非 git 降级**：检测不到 git 仓库时，明确警告无法 worktree 隔离，询问用户仍要串行派发还是放弃；不得静默继续。
```

- [ ] **Step 4: 重跑 S4 → 通过**

- [ ] **Step 5: Commit**

```bash
git add SKILL.md references/test-scenarios.md
git commit -m "feat: 派发流程与执行计划重写"
```

---

### Task 5: 验收流程

**Files:**
- Modify: `SKILL.md`（新增 `/wdp-prd-accept`）
- Modify: `references/test-scenarios.md`

**Interfaces:**
- Consumes: 派发流程产生的 worktree 分支、needs-review 状态。
- Produces: 验收流程；场景 S5。

- [ ] **Step 1: 写场景 S5（全生命周期）**

```markdown
## S5 全生命周期
- Setup: 单卡片 R-001 从 draft 开始，git 仓库。
- Action: 依次执行 confirm → dispatch → 模拟子代理返回报告 → accept。
- Expected:
  1. draft→confirmed→in-progress→needs-review→accepted→done 全程状态迁移正确。
  2. accept 流程包含：合并 worktree 分支、跑全量测试、code review、呈现给用户确认。
  3. done 后卡片「验收记录」段填写测试结果与 review 结论。
  4. 任一闸门（测试失败 / review 未过 / 用户未确认）→ 不进入 done，回到 needs-review。
```

- [ ] **Step 2: 干跑 S5 → 失败**

- [ ] **Step 3: 在 SKILL.md 增加 `/wdp-prd-accept`**

```markdown
### /wdp-prd-accept <id> —— 验收
1. 卡片 status==needs-review，否则提示先派发。
2. 合并子代理的 worktree 分支到当前工作分支。
3. 跑**全量集成测试**；失败 → 卡片留 needs-review，记录失败原因，提示修复或回滚。
4. 走 **code review**（复用 `superpowers:requesting-code-review`）。
5. 呈现变更摘要 + review 结论给用户 → 用户确认。
6. 确认 → status→accepted→done，填「验收记录」，重写 execution-plan.md。
7. 验收通过后清掉已合并的 worktree（复用 `superpowers:using-git-worktrees` 的清理流程）。
```

- [ ] **Step 4: 重跑 S5 → 通过**

- [ ] **Step 5: Commit**

```bash
git add SKILL.md references/test-scenarios.md
git commit -m "feat: 验收流程"
```

---

### Task 6: 错误处理 + 收尾验证

**Files:**
- Modify: `SKILL.md`（新增「错误处理」段 + 验证指令）
- Modify: `references/test-scenarios.md`
- Create: `scripts/README.md`（预留占位说明）

**Interfaces:**
- Consumes: 全部命令。
- Produces: 错误处理表 + 场景 S6 + 全量回归。

- [ ] **Step 1: 写场景 S6（错误分支）**

```markdown
## S6 错误分支
- Setup:
  - R-001 与 R-002 均 confirmed 且互相 conflicts_with（同批冲突）。
  - R-003 depends_on=[R-004]，R-004 depends_on=[R-003]（依赖成环）。
  - R-005 depends_on=[R-999]（依赖不存在）。
- Action: `/wdp-prd-dispatch` 全部。
- Expected:
  1. 同批冲突 → 提示只挑一个，不静默并行。
  2. 依赖成环 → 明确报告环（R-003↔R-004），不派发。
  3. 依赖不存在 → 报告「依赖 R-999 不存在，检查 depends_on」。
```

- [ ] **Step 2: 干跑 S6 → 失败**

- [ ] **Step 3: 在 SKILL.md 增加「错误处理」段 + 验证指令；写 scripts/README.md**

```markdown
## 错误处理
| 情况 | 处理 |
|------|------|
| 非 git 仓库 | 无法 worktree 隔离 → 明确警告，降级串行/非隔离（见派发流程） |
| 子代理失败 | 卡片 → blocked，报告原因，更新 execution-plan |
| 依赖成环 | 检测并报告环，不派发 |
| 依赖不存在 | 报告依赖 id 不存在，提示检查 depends_on |
| 同批冲突 | 强制只挑一个 |
| 无 Agent 工具 | 降级为手动引导流程 |
| `.wdp-prd` 缺失 | 触发初始化 |

## 验证
按 `references/test-scenarios.md` 干跑全部场景（S1–S6）。全部 Expected 满足才算 skill 验证通过。
```

```markdown
# scripts/ 预留目录

当前为空。v2 规划：Python/Node 调度脚本（拓扑排序、报告生成），见 spec §13 后续迭代 #3。
```

- [ ] **Step 4: 重跑 S6 → 通过；全量回归 S1–S6 全部通过**

- [ ] **Step 5: Commit**

```bash
git add SKILL.md references/test-scenarios.md scripts/README.md
git commit -m "feat: 错误处理、验证指令、scripts 占位"
```

---

## Self-Review 记录

**Spec 覆盖核对（对照 spec §1–§14）：**
- §3 存储模型（全局根 + 子目录 + 指针）→ Task 1 ✅
- §4 卡片模型 / §5 状态机 → Task 1 ✅
- §6 全部 7 个命令 → Task 2（prd/list/update）、Task 3（confirm）、Task 4（dispatch）、Task 5（accept）、Task 1（new 隐含于初始化后的建卡流程，独立在 Task 1 Step 5 验证）✅
- §7 调度算法 + 并发上限 + 同批约束 → Task 3 ✅
- §8 派发流程 / §9 验收流程 → Task 4 / Task 5 ✅
- §10 错误处理 → Task 6 ✅
- §11 测试（场景化演练）→ 全部任务场景 ✅
- §13 后续迭代 → scripts/README.md 记录 ✅

**Self-review 检查：**
- 无占位符：所有步骤含实际内容。
- 类型一致性：frontmatter 字段、状态机取值、命令名在所有任务中一致（`wdp-` 前缀、8 状态、5 priority 外的字段集合无漂移）。
- 已知缺口：`/wdp-prd-new` 未单独成任务——它在 Task 1 作为建卡流程实现（调用 superpowers:brainstorming → 存卡片 → draft），并在 S1 验证。已在范围说明中标注。

## Execution Handoff

计划已保存。执行方式二选一（见会话流程）。
