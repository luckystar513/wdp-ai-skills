---
name: wdp-list
description: Use when the user runs /wdp-list or asks to list the saved work snapshots and project profiles
disable-model-invocation: true
---

# /wdp-list — 列出所有总结文档

列出存储根目录下的 **profile（稳定层）** 和 **快照（工作状态）**，快照按时间从新到旧排列。

## 存储根目录（所有 wdp 命令共用）

1. 若本技能目录存在兄弟目录 `wdp-sum`（即 `../wdp-sum/summaries/`），优先使用它；
2. 否则回退到 `~/.claude/skills/wdp-sum/summaries/`。

## 执行步骤

### 1. 枚举
- 递归列出 `<存储根>/**/*.md`，排除 `latest.md`（指针文件）；
- 若为空，报告"暂无总结文档"，提示可用 `/wdp-sum` 生成第一份快照、`/wdp-profile` 生成档案。

### 2. 整理与展示
分两部分展示：

**A. 项目档案（profile）**
| 项目 | 更新时间 | 路径 |

**B. 工作快照（snapshot，新到旧）**
| 生成时间 | 项目 | 要点(标题) | 路径 |

- 快照按文件名时间戳从大到小排序（字典序即时间序）；
- 若快照有 frontmatter，可读 `created` 和第一行标题作为"要点"列；
- 末尾统计：共 N 份快照，M 份档案，分属 K 个项目。

### 3. 进一步动作
- 用户想看某份内容 → `/wdp-init <项目路径>` 读最新，或直接 Read 指定路径。
- 用户想更新稳定信息 → `/wdp-profile <项目路径>`。

## 常见错误

| 错误 | 修正 |
|------|------|
| 排序方向反了 | 新到旧，最新在最上面 |
| 把 profile.md/latest.md 混进快照列表 | 分开展示，latest.md 排除 |
| 只列一个项目 | 递归列出所有子目录 |
