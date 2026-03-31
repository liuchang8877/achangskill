---
name: mind_palace
description: 🏛️ 1:1 还原 Claude Code 源码级的层次化记忆宫殿 (INDEX.md + Topic Files)
metadata:
  openclaw:
    requires:
      bins: ["rg"]
---

# Claude Code 1:1 记忆系统 (Memory System)

这是一个高级记忆管理技能,用于记录和回溯对话中的"暗物质"信息(无法从代码库中推导出的背景)。

## 1. 记忆存储结构
所有记忆持久化在 `.openclaw/memory/` 目录下:
- **`INDEX.md`**: 200 行以内的极简摘要索引。
- **`topics/`**: 详细的主题信息 Markdown 文件。

## 2. 判别准则 (什么是值得记忆的?)
- **拒绝规则**: 凡是能通过 `grep`、`git log` 或读取源文件查出的事实,一律不计入记忆。
- **包含准则**:
    - **User**: 用户的角色、专业深度、沟通风格。
    - **Feedback**: 习惯修正(必须记下 Why 和 How to apply)。
    - **Project**: 外部截止日期、合规背景、不明显的架构妥协。
    - **Reference**: 外部链接(Linear, Slack, Grafana)。

## 3. 使用工具流 (Step-by-Step)
### 检索 (Retrieval)
1. 始终优先预览 `INDEX.md`。
2. 发现相关线索后,调用 `read_file` 读取 `topics/` 下的文件。
3. **验证**: 使用记忆前,必须验证相关实体(文件/函数)是否在当前代码库中依然存在。

### 主动记录 (Active Save)
当用户明确要求"记住"或触发了严重反馈时:
1. **Step 1**: 写入 `topics/主题.md`(带 YAML Frontmatter)。
2. **Step 2**: 在 `INDEX.md` 末尾追加一行指针:`- [标题](topics/文件.md) - 简要摘要`。

### 自动清理与总结 (Purge & Summarize)
当 `INDEX.md` 接近 200 行时,主动提议合并相似话题或删除过期记忆。

## 4. 被动捕获模式 (Passive Capture)
除了用户明确要求"记住"之外，AI 应在每次 heartbeat 或对话结束时主动检查：
- 是否有新的用户反馈/纠正需要记录？
- 是否有新的项目状态变化？
- 是否有新的偏好或规则被表达？

如果有，即使用户没有说"记住"，也要自动写入 `topics/` 并更新 INDEX。

## 5. 后台提取提示词 (针对 Extraction Agent)
请扮演提取分身，分析刚才的对话流。提取那些"令人惊讶"或"非显而易见"的信息。重点关注用户纠正你的时刻，并提炼其背后的规则。
