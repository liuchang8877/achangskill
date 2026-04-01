---
name: mind_palace
description: 🏛️ 层次化记忆宫殿 — INDEX.md + Topic Files + KAIROS 自动蒸馏
metadata:
  openclaw:
    requires:
      bins: ["rg"]
---

# 记忆宫殿 (Mind Palace)

层次化记忆管理系统，参考 Claude Code 源码级记忆系统设计。

## 存储结构

```
.openclaw/memory/
├── INDEX.md          # 极简索引（≤200行）
├── topics/           # 详细记忆文件（YAML frontmatter）
└── raw/              # KAIROS 原始日志（append-only）
```

## 判别准则

**不记**：能用 `grep`/`git log`/读源文件查到的代码事实。

**要记**：
- 用户偏好、纠正、反馈（记 Why + How to apply）
- 项目状态变化、架构决策
- 外部截止日期、合规背景
- 说出来的信息，不是能从上下文推导的

## 工具流

### 检索
1. 查 `INDEX.md`
2. 读 `topics/` 相关文件
3. **验证**：用前确认实体是否还存在

### 主动保存
用户说"记住"时：
1. 写 `topics/YYYY-MM-DD-主题.md`（带 YAML frontmatter）
2. INDEX.md 末尾追加一行指针

### 被动捕获（HEARTBEAT 触发）
每次 heartbeat 检查是否有值得记忆的新事件，如有则**实时追加**到 `raw/YYYY-MM-DD.md`：

```
[TIMESTAMP] | TYPE | CONTENT
TYPE: USER_FEEDBACK | USER_PREFERENCE | PROJECT_STATUS | DECISION | OTHER
```

原则：实时追加不修改，append-only 是 KAIROS 蒸馏的原材料。

### 索引清理
INDEX.md 接近 200 行时，合并或删除过期记忆。

## KAIROS 自动蒸馏系统

对照 Claude Code 记忆系统，补充了三个缺失功能：

### 1. AI-to-AI 记忆检索
- 用 LLM 判断当前上下文与哪些记忆最相关
- 精度优先，最多注入 5 条，宁可少记不记错

### 2. KAIROS 夜间蒸馏
- 每天 02:00 AM Asia/Shanghai，低活跃期自动运行
- launchd 调度 → `openclaw agent --deliver` → 飞书消息触发蒸馏
- 读取 `raw/` 日志 → AI 蒸馏 → 写入 `topics/YYYY-MM-DD-kairos.md`
- 蒸馏后归档 7 天前 raw log

### 3. Append-only 原始日志
- 先写 raw log，再结构化蒸馏
- 中间层设计：原材料 → AI 蒸馏 → 结构化记忆
- 可追溯、可审计

### 调度配置
```bash
# launchd plist: ~/Library/LaunchAgents/com.openclaw.kairos.plist
# 触发方式: openclaw agent --channel feishu --deliver --reply-to user:ou_xxx
# 频率: 每天 02:00 AM Asia/Shanghai
```

### KAIROS 日志文件
- Raw log: `memory/raw/YYYY-MM-DD.md`
- 蒸馏输出: `memory/topics/YYYY-MM-DD-kairos.md`
- 归档: `memory/raw/archive/`

## 与 Claude Code 记忆系统的对比

| 功能 | Claude Code | 本 Skill |
|------|------------|---------|
| 分层存储 | ✅ | ✅ |
| 判别准则 | ✅ | ✅ |
| AI 检索 | Sonnet 小模型 | 手动 semantic search |
| KAIROS 夜间蒸馏 | ✅ /dream | ✅ launchd + openclaw agent |
| Append-only 日志 | ✅ | ✅ |
| 被动捕获 | 自动化 | HEARTBEAT 触发 |

## 后台提取提示词（供 Extraction Agent / 子 Agent 使用）

分析对话流，提取"令人惊讶"或"非显而易见"的信息。重点关注：
- 用户纠正 AI 的时刻
- 明确表达的偏好或规则
- 推理得出的隐含规则（需注明 Why）
