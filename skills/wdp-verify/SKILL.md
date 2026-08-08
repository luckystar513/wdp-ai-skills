---
name: wdp-verify
description: Use when the user runs /wdp-verify, or asks to check whether the saved work snapshot is still accurate/stale compared to the current code before trusting it for continuation
disable-model-invocation: true
---

# /wdp-verify — 校验快照与代码漂移

**只读**操作：重新读取最新快照 + profile，与**当前代码状态**对比，报告**漂移（drift）**——快照里的"进行中 / 下一步 / 文件 / 提交"哪些已过时、哪些仍成立，并给出"该不该刷新快照"的判断。

**触发时机**：接续前想确认快照是否可信、`/wdp-init` 读到的东西和代码对不对得上。

## 存储根目录（所有 wdp 命令共用）

1. 若本技能目录存在兄弟目录 `wdp-sum`（即 `../wdp-sum/summaries/`），优先使用它；
2. 否则回退到 `~/.claude/skills/wdp-sum/summaries/`。

## 执行步骤

### 1. 确定项目
- 无参数：当前工作目录（`pwd`）。
- 有参数：`/wdp-verify <项目路径>`。

### 2. 读取快照（只读）
- 读 `latest.md` 指向的最新快照（或按时间戳取最新）；无快照则报告"尚无快照，先运行 /wdp-sum"并结束。
- 解析 frontmatter：`created`、`commit`、`prev`、`profile`。

### 3. 采集当前状态（只读探索，不修改任何文件）
- **git**：`git status --short`、`git branch --show-current`、`git log --oneline -5`、`git diff --stat`（不在仓库则跳过 git 项）。
- **文件存在性**：快照与 profile 中提到的关键路径（入口、模块目录、配置文件）逐一检查是否仍存在。
- **任务项**：快照"进行中 / 下一步"里的条目，对照当前代码判断是否已达成、是否已无意义。

### 4. 逐项判定漂移（核心步骤）
对每条做 4 类判定：

| 类别 | 判定 | 含义 |
|------|------|------|
| ✅ 仍成立 | 与当前状态一致 | 无需处理 |
| 🔄 有进展未记录 | git 有新提交/改动，快照未反映 | 快照过期，建议 /wdp-sum |
| ⚠️ 已失效 | 快照提到的文件/模块/下一步已不存在或已完成 | 条目作废，需 /wdp-sum 修订 |
| ❌ 冲突 | 快照与代码矛盾（如快照说"用 A 方案"而代码已改 B） | 立即刷新，避免按旧快照接续 |

- **commit 对比**：快照 `commit` 与当前 HEAD 不一致 → 明确提示"快照基于 X 提交，当前为 Y"。
- **profile 层**：若发现架构/技术栈/规则在代码里已明显变化（如新增框架、目录重构），标注"profile 需 /wdp-profile 刷新"。

### 5. 报告（不修改存储）
- 快照时间 vs 当前时间、快照 commit vs 当前 HEAD；
- 漂移表（上述 4 类的逐条结果）；
- **陈旧度结论**：新鲜（可直接接续）/ 略陈旧（参考但建议刷新）/ 严重过期（必须先 /wdp-sum 或 /wdp-profile 再继续）；
- 建议动作：`/wdp-sum`（状态刷新）或 `/wdp-profile`（稳定信息刷新）。

## 常见错误

| 错误 | 修正 |
|------|------|
| verify 时改文件 | verify 是只读校验，任何写入都不做 |
| 只对比 git 不查文件存在性 | 快照提到的路径可能已被删/改名，必须逐一核对 |
| 把"有进展"当"冲突" | 有进展 = 快照过期建议刷新；冲突 = 内容矛盾要立即处理，两者结论不同 |
| 忽略 profile 变化 | 稳定层漂移要靠 /wdp-profile，不是 /wdp-sum |
