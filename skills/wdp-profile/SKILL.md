---
name: wdp-profile
description: Use when the user runs /wdp-profile or asks to create/refresh the project's stable profile (tech stack, architecture, rules, run instructions) that persists across sessions
disable-model-invocation: true
---

# /wdp-profile — 维护项目稳定层

就地创建或更新项目的 `profile.md` —— 承载**变化缓慢、必须长期遵守**的项目知识：技术栈、架构、业务领域、**不可违背的规则/不变量**、运行方式、外部依赖、决策记录、已知坑。

**与 /wdp-sum 的分工**：`/wdp-profile` 管"长期稳定信息"，`/wdp-sum` 管"当前工作状态快照"。快照引用 profile，profile 不重复快照内容。

## 存储位置

```
<存储根>/<项目slug>/profile.md
```

存储根解析同其他 wdp 命令：优先 `../wdp-sum/summaries/`，否则 `~/.claude/skills/wdp-sum/summaries/`。

## 执行步骤

### 1. 确定项目
- 无参数：当前工作目录（`pwd`）。
- 有参数：`/wdp-profile <项目路径>`。

### 2. 收集稳定信息（只读探索）
- **技术栈**：清单文件（`package.json`、`requirements.txt`、`pyproject.toml`、`go.mod`、`Cargo.toml` 等）+ 配置（`vite.config.*`、`tsconfig.json`、`.env.example`、`docker-compose.yml` 等）→ 归纳语言/框架/关键依赖。
- **架构**：目录结构、模块边界、入口点、数据流；读 `docs/`、架构相关文档。
- **业务/领域**：README、项目用途、目标、命名 → 归纳业务领域。
- **运行方式**：`package.json` scripts、Makefile、README 里的 build/test/run/部署命令。
- **规则/不变量**：读现有 `CLAUDE.md`、`AGENTS.md`、`CONTRIBUTING.md`、`.editorconfig`、代码里的约束注释 → 提取**必须遵守、不可违背**的规则（写进专门的"不可违背的规则"节，与临时注意事项严格区分）。
- **决策记录**：README/CHANGELOG/docs 里记录的架构决策及其理由。
- **已知坑**：从注释、issue、历史提交、常见报错中归纳。
- **外部依赖**：第三方服务、API、凭据位置（只记位置，不记密钥）。

### 3. 生成或更新 profile.md
- 首次：按模板生成。
- 已存在：**逐节对比**，只更新有变化的节；删除已失效的"坑"；保留并修订"规则"。
- frontmatter 的 `updated` 更新为当前时间。

## profile 模板

```markdown
---
type: profile
project: <项目slug>
updated: <YYYY-MM-DDTHH:MM:SS>
---

# 项目档案 · <项目名>

## 一句话定位
<一句话：这个项目是什么>

## 技术栈
<语言 / 框架 / 关键依赖>

## 架构概览
<模块边界、入口、数据流，3-8 行>

## 业务领域
<做什么、给谁用、核心价值>

## 不可违背的规则（Invariants）
- <必须遵守的规则/不变量，禁止违反>
- <例如：不得直接改数据库结构；所有对外 API 必须向后兼容…>

## 运行方式
- 构建：<命令>
- 测试：<命令>
- 运行：<命令>
- 部署：<命令>

## 外部依赖
- <服务/API/凭据位置>

## 决策记录（Decisions & why）
- <What + Why>

## 已知坑（Known gotchas）
- <坑 + 规避方法>
```

### 4. 报告结果
- profile.md 路径；
- 新增/更新了哪些节；
- 明确列出"不可违背的规则"节内容，请用户确认是否准确。

## 常见错误

| 错误 | 修正 |
|------|------|
| 把当前任务状态写进 profile | 那是快照的职责，profile 只放长期稳定信息 |
| 规则和坑混在一起 | 规则=不可违背，坑=规避建议，分开写 |
| 复制密钥/凭据 | 只记位置，不记密钥 |
