---
name: wdp-init
description: Use when the user runs /wdp-init, or at the start of a session after context was cleared/compacted, to load the project's saved profile and latest work snapshot
disable-model-invocation: true
---

# /wdp-init — 载入项目上下文

读取项目的 **profile（稳定层）+ 最新快照（工作状态）**，把技术、业务、规则、当前工作载入当前对话，让本次会话（或刚启动的 agent）快速接续。

## 存储模型

```
<存储根>/<项目slug>/
├── profile.md            # 稳定层（/wdp-profile 维护）
├── <时间戳>.md           # 快照层（/wdp-sum 追加）
└── latest.md             # 指向最新快照
```

## 存储根目录（所有 wdp 命令共用）

1. 若本技能目录存在兄弟目录 `wdp-sum`（即 `../wdp-sum/summaries/`），优先使用它；
2. 否则回退到 `~/.claude/skills/wdp-sum/summaries/`。

## 执行步骤

### 1. 确定项目
- 无参数：当前工作目录（`pwd`）。
- 有参数：`/wdp-init <项目路径>`。

### 2. 读取 profile（稳定层）
- 读 `<项目slug>/profile.md`；
- 若不存在：报告"尚无 profile，建议运行 /wdp-profile"，继续读快照；
- 存在：完整 Read，向用户复述要点：一句话定位、技术栈、架构、**不可违背的规则**、运行方式、已知坑。

### 3. 读取最新快照（工作状态）
- 若存在 `latest.md`：读取其指向的快照；
- 否则：按文件名时间戳排序取最新 `.md`（排除 `profile.md`、`latest.md`）；
- 完整 Read 该快照，向用户复述：当前焦点、自上次完成、进行中、下一步、非显然知识、决策。
- 若项目目录为空：回退到 `<存储根>/` 下所有子目录里的最新快照，并**明确告知**"读取到的是另一个项目的总结"。

### 4. 载入完成
- 说明读取的 profile 路径 + 快照路径 + 时间；
- 根据"进行中/下一步"，主动给出 2-3 个建议切入点，询问用户继续哪项。

## 会话开始约定（自动接续）

在本会话**后续任何时候**，若发生以下情况，主动提醒用户保存当前工作状态：

- 会话上下文即将被**压缩 / 清空**（用户要求 /clear，或明显上下文过长）；
- 完成了**重大改动**或一个**里程碑**；
- 即将**切换 agent / 关闭会话 / 交接他人**。

此时应主动建议："建议运行 `/wdp-sum` 保存当前快照，下次可直接 `/wdp-init` 接续。"

> 此约定让 wdp 从"手动工具"变成"自动记忆"：agent 在关键时刻自己提出来，而不是等用户想起。

## 常见错误

| 错误 | 修正 |
|------|------|
| 只读快照不读 profile | 两层都要读，规则/技术栈在 profile 里 |
| 把 profile.md/latest.md 当快照排序 | 排序时排除这两个文件 |
| 读错项目的文档 | 先按当前项目名定位子目录 |
| 只读文件名不读内容 | 必须完整 Read，真正载入上下文 |
