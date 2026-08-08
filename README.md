# wdp-ai-skills — Coding Agent 技能集合

一系列面向 coding agent 的 Agent Skills（遵循 [agentskills.io](https://agentskills.io) 规范）。**`skills/` 下每个目录 = 一个独立技能**，可单独安装，也可作为插件整体分发。本仓库会持续收录更多技能。

> **中英双版**：每个技能都提供全中文（`<name>`）与全英文（`<name>-en`）两个版本，共用存储、结构一致，国内/国外用户各取所需。

## 目录

| 技能 | 功能 |
|------|------|
| [wdp-ctx](skills/wdp-ctx/SKILL.md) | 跨会话/跨 agent 的项目上下文：随时保存、随时接续、随时切换（sum/init/profile/verify/export/list/clear 七个子命令）。中文版 |
| [wdp-ctx-en](skills/wdp-ctx-en/SKILL.md) | Same as wdp-ctx — the English variant (shared storage, switch language without losing context) |

---

## wdp-ctx — 跨会话/跨 agent 项目上下文保留

一条命令，把当前项目的工作状态固化成带时间戳的快照，让任何 coding agent 在重启、清空、压缩上下文后随时接续上次工作。**纯 Markdown 技能，无运行时代码。**

### 一条命令，七个子命令

```
/wdp-ctx <子命令> [参数]
```

| 子命令 | 功能 | 参数 |
|--------|------|------|
| `sum` | 固化当前工作状态为快照（增量对比上一次，✅🔄⛔🔀 标注进展） | `<项目路径>` `--deep` |
| `init` | 读取 profile + 最新快照，把项目全貌载入当前对话，快速接续 | `<项目路径>` |
| `profile` | 创建/更新项目**稳定层**：技术栈、架构、业务、不可违背规则、运行方式、已知坑 | `<项目路径>` |
| `verify` | 校验最新快照与当前代码的**漂移**（git 对比、文件存在性、条目过时判定） | `<项目路径>` |
| `export` | 把 profile + 快照合成为 **AGENTS.md**（标记维护区），让不认 `/wdp-ctx` 的其他 agent 启动即读 | `<项目路径>` |
| `list` | 列出所有文档（profile 档案 + 快照，新到旧） | — |
| `clear` | 清空快照（默认保留 profile；支持 `--keep N` 保留最新 N 份；删除前二次确认） | `<项目路径>` `--keep N` `--profile` |

无子命令直接运行 → 默认 `list` 并展示用法。`--deep` 为显式 opt-in：派生子 agent 按子系统深度分析。

### 核心模型：两层文档

```
<存储根>/<项目slug>/
├── profile.md            # 稳定层：技术栈/架构/规则/运行方式（长期不变，profile 子命令维护）
├── <YYYY-MM-DD_HHMMSS>.md  # 快照层：当前焦点/完成/进行中/下一步/非显然知识（sum 子命令追加）
└── latest.md             # 指针：指向最新快照（init 读它）
```

**稳定信息归 profile，易变信息归快照** —— 快照不重复粘贴技术栈，只记录状态变化，通过 frontmatter 的 `profile:` 引用稳定层。

### 安装（Claude Code）

把 `skills/wdp-ctx` 与 `skills/wdp-ctx-en` 复制到 `~/.claude/skills/`（两版共用存储，装任意一版即可，也可都装）：

```bash
mkdir -p ~/.claude/skills
cp -r skills/wdp-ctx skills/wdp-ctx-en ~/.claude/skills/
```

安装后重启 Claude Code（或稍等热加载），即可在任何项目使用 `/wdp-ctx`（中文版）或 `/wdp-ctx-en`（英文版）。

> 用户级安装对**所有项目生效**。若只想在单个项目内生效，把对应目录复制到该项目的 `.claude/skills/` 即可。

### 使用流程

1. **首次**：`/wdp-ctx profile` → 建立项目稳定档案（技术栈/规则/运行方式）。
2. **开工前**：`/wdp-ctx init` → 读取档案 + 最新快照，快速接续；拿不准快照是否过时先 `/wdp-ctx verify`。
3. **收工/里程碑**：`/wdp-ctx sum` → 固化当前状态，生成增量快照。
4. **交接其他 agent**：`/wdp-ctx export` → 生成项目根目录 `AGENTS.md`，对方启动即读。
5. 随时 `/wdp-ctx list` 查看历史；`/wdp-ctx clear --keep 5` 清理旧快照但保留最近 5 份。

### 存储位置

所有子命令共用存储根（按 **项目 slug 分子目录**、文件名含时间戳，多项目互不混淆）：

- 用户级安装：`~/.claude/skills/wdp-ctx/summaries/`
- 项目级安装：`<项目>/.claude/skills/wdp-ctx/summaries/`
- 解析规则：项目级优先，用户级兜底。

### 自动提醒（可选，hooks）

技能内置"关键时刻主动建议"约定：agent 会在上下文压缩/清空、里程碑完成、会话结束前主动提醒你 `/wdp-ctx sum`。若要更硬的兜底，可挂会话级 hooks（非阻塞，仅当项目已有快照才提示）：

```jsonc
// .claude/settings.json 或 settings.local.json
{
  "hooks": {
    "SessionStart": [{ "hooks": [{ "type": "command", "command": "bash \"<路径>/wdp-hook.sh\" load" }] }],
    "PreCompact":  [{ "matcher": "manual|auto", "hooks": [{ "type": "command", "command": "bash \"<路径>/wdp-hook.sh\" save-precompact" }] }],
    "Stop":        [{ "hooks": [{ "type": "command", "command": "bash \"<路径>/wdp-hook.sh\" save" }] }]
  }
}
```

`wdp-hook.sh` 见 [skills/wdp-ctx/](skills/wdp-ctx/) 的 hooks 安装示例。

---

## 新增技能

每个技能**中英双版**，平铺在 `skills/` 下（**不要嵌套**——Claude Code 不发现嵌套技能目录）：

```
skills/<技能名>/             # 全中文（命令 /<技能名>）
├── SKILL.md
└── summaries/             # 可选：技能自有存储（如 wdp-ctx 的共享存储根）
skills/<技能名>-en/         # 全英文（命令 /<技能名>-en）
└── SKILL.md
```

SKILL.md 要求：
- frontmatter `name` 与目录名一致、`description` 以 "Use when" 开头只写触发条件、`disable-model-invocation: true`
- 纯 Markdown 优先，无运行时代码；需脚本时放同目录并给出用法
- 中英两版命令/子命令/模板结构完全一致，只差语言；若共用存储，双方 SKILL.md 存储节互注说明

新增后更新本 README 目录表与 `.claude-plugin/plugin.json` 描述。

## 插件分发

本仓库含 `.claude-plugin/plugin.json`，可整体作为 Claude Code 插件使用。校验后推送 git 仓库即可：

```bash
claude plugin validate .
```

注意：插件形式命令带命名空间前缀（如 `/wdp-ctx:wdp-ctx`），裸命令 `/wdp-ctx` 需用户级 skills 安装。

## 兼容性

- **Claude Code**：标准 skills 机制，开箱即用。
- **其他 coding agent**：纯 Markdown + 标准目录结构，遵循 Agent Skills 规范（agentskills.io），支持 skills 目录的 agent（如 Codex、Gemini CLI 的 `~/.agents/skills/` 别名）可直接复用；不支持的 agent 用 `/wdp-ctx export` 生成的 `AGENTS.md` 接续。
