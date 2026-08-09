# wdp-ai-skills — Coding Agent 技能集合

一系列面向 coding agent 的 Agent Skills（遵循 [agentskills.io](https://agentskills.io) 规范）。**`skills/` 下每个目录 = 一个独立技能**，可单独安装，也可作为插件整体分发。本仓库会持续收录更多技能。

每个技能提供中英两个**独立版本**，用户**二选一安装**：
- **中文用户** → 安装 [wdp-ctx](skills/wdp-ctx/README.md)（全中文）
- **英文用户** → 安装 [wdp-ctx-en](skills/wdp-ctx-en/README.md)（全英文）

两版各自独立发布（各自的 README / 更新记录 / hook 脚本），共用存储根（中英切换不丢上下文）。

## 技能

| 技能 | 语言 | 功能 | 文档 |
|------|------|------|------|
| [wdp-ctx](skills/wdp-ctx/) | 中文 | 跨会话/跨 agent 项目上下文保留：sum/init/profile/verify/export/list/clear 七个子命令 | [README](skills/wdp-ctx/README.md) |
| [wdp-ctx-en](skills/wdp-ctx-en/) | English | Same features — English variant | [README](skills/wdp-ctx-en/README.md) |

## 安装（二选一）

- **中文用户**：`cp -r skills/wdp-ctx ~/.claude/skills/`（详见 [wdp-ctx README](skills/wdp-ctx/README.md)）
- **英文用户**：`cp -r skills/wdp-ctx-en ~/.claude/skills/`（见 [wdp-ctx-en README](skills/wdp-ctx-en/README.md)）
- **插件方式**：整个仓库作为 Claude Code 插件安装，含两版 + 自动注册 hooks（默认中文脚本）。

## 新增技能

每个技能**中英双版**，平铺在 `skills/` 下（**不要嵌套**——Claude Code 不发现嵌套技能目录），每版自带独立 README 与可选 hooks：

```
skills/<技能名>/             # 全中文（命令 /<技能名>）
├── SKILL.md
├── README.md              # 独立文档 + 更新记录
└── hooks/                 # 技能自己的 hook 脚本（可选）
skills/<技能名>-en/         # 全英文（命令 /<技能名>-en）
├── SKILL.md
├── README.md
└── hooks/
```

SKILL.md 要求：frontmatter `name` 与目录名一致、`description` 以 "Use when" 开头只写触发条件、`disable-model-invocation: true`。新增后更新本 README 目录表与 `.claude-plugin/plugin.json` 描述。

## 插件分发

本仓库含 `.claude-plugin/plugin.json`（含 `"hooks": "./hooks/hooks.json"`），可整体作为 Claude Code 插件使用。校验后推送 git 仓库即可：

```bash
claude plugin validate .
```

注意：插件形式命令带命名空间前缀（如 `/wdp-ctx:wdp-ctx`），裸命令 `/wdp-ctx` 或 `/wdp-ctx-en` 需用户级 skills 安装。

## 兼容性

- **Claude Code**：标准 skills 机制，开箱即用。
- **其他 coding agent**：纯 Markdown + 标准目录结构，遵循 Agent Skills 规范（agentskills.io）；不支持的 agent 用 `/wdp-ctx export` 生成的 `AGENTS.md` 接续。
