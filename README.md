# wdp-sum — 跨会话/跨 agent 项目上下文保留

把当前项目的工作状态固化成带时间戳的快照，让任何 coding agent 在重启、清空、压缩上下文后，用一条命令完全接续上次工作。**纯 Markdown 技能，无运行时代码。**

## 七个命令

| 命令 | 功能 |
|------|------|
| `/wdp-sum` | 固化当前工作状态为快照（增量对比上一次，✅🔄⛔🔀 标注进展；`--deep` 派生子 agent 深度分析） |
| `/wdp-init` | 读取 profile + 最新快照，把项目全貌载入当前对话，agent 快速接续 |
| `/wdp-profile` | 创建/更新项目**稳定层**：技术栈、架构、业务、不可违背规则、运行方式、已知坑 |
| `/wdp-verify` | 校验最新快照与当前代码的**漂移**（git 对比、文件存在性、条目过时判定），判断快照是否可信 |
| `/wdp-export` | 把 profile + 快照合成为 **AGENTS.md**（标记维护区），让不认 `/wdp-*` 的其他 agent 启动即读 |
| `/wdp-list` | 列出所有文档（profile 档案 + 快照，新到旧） |
| `/wdp-clear` | 清空快照（默认保留 profile；支持 `--keep N` 保留最新 N 份；删除前二次确认） |

## 核心模型：两层文档

```
<存储根>/<项目slug>/
├── profile.md            # 稳定层：技术栈/架构/规则/运行方式（长期不变，/wdp-profile 维护）
├── <YYYY-MM-DD_HHMMSS>.md  # 快照层：当前焦点/完成/进行中/下一步/非显然知识（/wdp-sum 追加）
└── latest.md             # 指针：指向最新快照（/wdp-init 读它）
```

**稳定信息归 profile，易变信息归快照** —— 快照不重复粘贴技术栈，只记录状态变化，通过 frontmatter 的 `profile:` 引用稳定层。

## 目录结构

```
wdp-sum/
├── README.md
├── skills/                       # ← 复制这 7 个子目录到 ~/.claude/skills/
│   ├── wdp-sum/SKILL.md          #   → /wdp-sum
│   │   └── summaries/            #     存储根（运行时生成，七个命令共用）
│   ├── wdp-init/SKILL.md         #   → /wdp-init
│   ├── wdp-profile/SKILL.md      #   → /wdp-profile
│   ├── wdp-verify/SKILL.md       #   → /wdp-verify
│   ├── wdp-export/SKILL.md       #   → /wdp-export
│   ├── wdp-list/SKILL.md         #   → /wdp-list
│   └── wdp-clear/SKILL.md        #   → /wdp-clear
```

## 安装（Claude Code）

把 `skills/` 下的 7 个目录复制到 `~/.claude/skills/`：

```bash
mkdir -p ~/.claude/skills
cp -r skills/wdp-sum skills/wdp-init skills/wdp-profile skills/wdp-verify \
      skills/wdp-export skills/wdp-list skills/wdp-clear ~/.claude/skills/
```

安装后重启 Claude Code（或稍等热加载），即可在任何项目使用 `/wdp-*` 命令。

> 用户级安装对**所有项目生效**。若只想在单个项目内生效，把 `skills/` 下目录复制到该项目的 `.claude/skills/` 即可。**7 个技能必须作为兄弟目录装在一起**（存储依赖相对定位，单独安装会回退到用户级存储根）。

## 使用流程

1. **首次**：`/wdp-profile` → 建立项目稳定档案（技术栈/规则/运行方式）。
2. **开工前**：`/wdp-init` → 读取档案 + 最新快照，快速接续；拿不准快照是否过时先 `/wdp-verify`。
3. **收工/里程碑**：`/wdp-sum` → 固化当前状态，生成增量快照。
4. **交接其他 agent**：`/wdp-export` → 生成项目根目录 `AGENTS.md`，对方启动即读。
5. 随时 `/wdp-list` 查看历史；`/wdp-clear --keep 5` 清理旧快照但保留最近 5 份。

## 自动提醒（可选，hooks）

技能内置"关键时刻主动建议"约定：agent 会在上下文压缩/清空、里程碑完成、会话结束前主动提醒你 `/wdp-sum`。若要更硬的兜底，可挂会话级 hooks（非阻塞，仅当项目已有快照才提示）：

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

`wdp-hook.sh` 见本仓库 `.claude/wdp-hook.sh`（含存储根解析与"仅项目有快照才提醒"的条件逻辑）。

## 存储位置

所有命令共用存储根（按 **项目 slug 分子目录**、文件名含时间戳，多项目互不混淆）：

- 用户级安装：`~/.claude/skills/wdp-sum/summaries/`
- 项目级安装：`<项目>/.claude/skills/wdp-sum/summaries/`
- 解析规则：技能目录存在兄弟 `wdp-sum`（`../wdp-sum/summaries/`）则用它，否则回退用户级。

## 上传到 skills 网站 / marketplace

技能文件已按标准 Agent Skills 格式组织（`<name>/SKILL.md` + YAML frontmatter `name`/`description`），可直接作为 skill bundle 上传。打包成 Claude Code 插件时：

```
wdp-sum/
├── .claude-plugin/plugin.json    # name/description（已含 7 命令）
└── skills/                       # 插件直接引用
```

用 `claude plugin validate .` 校验后推送 git 仓库并挂 marketplace。注意：插件形式命令带命名空间前缀（如 `/wdp-sum:wdp-sum`），裸命令 `/wdp-*` 需用户级 skills 安装。

## 兼容性

- **Claude Code**：标准 skills 机制，开箱即用。
- **其他 coding agent**：纯 Markdown + 标准目录结构，遵循 Agent Skills 规范（agentskills.io），支持 skills 目录的 agent（如 Codex、Gemini CLI 的 `~/.agents/skills/` 别名）可直接复用；不支持的 agent 用 `/wdp-export` 生成的 `AGENTS.md` 接续。
