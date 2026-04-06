---
name: mind_palace
description: 🏛️ LLM Wiki 模式记忆系统 — 实时 Ingest + 批量蒸馏双轨制
metadata:
  openclaw:
    requires:
      bins: ["rg"]
---

# 记忆宫殿 (Mind Palace) — LLM Wiki 模式

参考 Claude Code 源码级记忆系统设计，采用 LLM Wiki 模式：知识一次编译，持续保持最新。

## 存储结构

```
.openclaw/workspace/
├── MEMORY.md              # Wiki 层 — LLM 编译后的知识（按 topic 组织）
├── HEARTBEAT.md           # 记忆系统工作流定义
├── AGENTS.md              # Workspace 运作规范
├── memory/
│   ├── INDEX.md           # Wiki 目录（按 category 组织）
│   ├── raw/               # 原料层 — append-only 原始日志
│   │   └── YYYY-MM-DD.md
│   └── archive/           # 归档的旧 raw log
└── scripts/
    └── kairos_distill.py  # 批量蒸馏脚本
```

## 双轨制

### 轨道一：实时 Ingest（每次对话后）
对话结束后立即将新知识编译进 MEMORY.md：
- 新 topic → 追加新 section
- 已有 topic 的补充/修正 → 原地更新
- 新信息推翻旧认知 → 直接修改原文，标注更新时间 + 来源
- 每个 entry 标注：`[YYYY-MM-DD HH:mm]` 时间戳

### 轨道二：批量蒸馏（每天 02:00 AM，launchd 触发）
- 脚本：`scripts/kairos_distill.py`
- 读取近 7 天 raw log
- LLM 整合 + Lint（矛盾检测、过时标记、孤儿条目）
- 原地更新 MEMORY.md 对应 section
- 归档 7 天前 raw log

两者区别：
| | 实时 Ingest | 批量蒸馏 |
|---|---|---|
| 时机 | 每次对话后 | 每天 2AM |
| 内容 | 单次对话新知识 | 全量 raw log 整合 |
| 操作 | 增量更新 MEMORY.md | 合并 + lint + 归档 |

## 判别准则

**不记**：
- 能用 `grep`/`git log`/读源文件查到的代码事实
- 临时性问答
- 闲聊和问候
- 明显的玩笑

**要记**：
- 用户明确纠正或批评 AI 的时刻
- 用户表达的偏好、规则、习惯
- 项目状态变化、架构决策
- 重要结论或决定
- 有价值的答案（用户问我，我回答了，这个回答值得沉淀）
- 任何"说出来而不是能从代码推断出来"的信息

## Wiki 层操作

### Ingest（对话后）
1. 扫描本次对话是否有值得记忆的新信息
2. 确定归属的 section（User / Architecture / Projects / Tool Setup / Reference / Patterns）
3. 写入 MEMORY.md（原地更新，不重复累积）
4. 追加 raw log 到 `memory/raw/YYYY-MM-DD.md`
5. 可选更新 INDEX.md

### Query（回答前）
1. 先读 MEMORY.md 找相关 page
2. 综合回答，必要时引用 MEMORY.md 中的 entry
3. 如果回答本身有价值（分析、对比、洞察），**主动问用户要不要归档**

### Lint（定期）
- 检查 MEMORY.md 内部是否有矛盾
- 标记过时信息（对比 raw log 中更新的记录）
- 找出孤儿条目（没有任何交叉引用的独立 page）
- 建议新的 wiki 页面或 topic

## Raw Log 格式

```
[HH:mm] | TYPE | CONTENT
TYPE: USER_FEEDBACK | USER_PREFERENCE | PROJECT_STATUS | DECISION | OTHER
```

原则：实时追加不修改，append-only 是 KAIROS 蒸馏的原材料。

## MEMORY.md Section 参考

| Section | 内容 |
|---------|------|
| ## 用户 (User) | LIUCHANG 基础信息、偏好、反馈 |
| ## 系统架构 (Architecture) | OpenClaw 部署结构、KAIROS 系统 |
| ## 项目 (Projects) | 具体项目状态、架构决策 |
| ## 工具配置 (Tool Setup) | opencli、agent-reach、fxtwitter 等 |
| ## 参考链接 (Reference) | X/Twitter 分享的技术文章 |
| ## 设计模式 (Patterns) | 工作流、设计模式 |
| ## 待办 & 问题 (Todos & Issues) | 未解决的问题和待办 |

## 调度配置

```bash
# launchd plist: ~/Library/LaunchAgents/com.openclaw.kairos.plist
# 触发方式: openclaw agent --channel feishu --deliver --reply-to user:ou_xxx
# 频率: 每天 02:00 AM Asia/Shanghai
# 脚本: scripts/kairos_distill.py
```

## 与 Claude Code 记忆系统的对比

| 功能 | Claude Code | 本 Skill |
|------|------------|---------|
| LLM Wiki 模式 | ✅ /dream | ✅ 双轨制 |
| 分层存储 | ✅ | ✅ |
| 判别准则 | ✅ | ✅ |
| 实时 Ingest | ✅ | ✅ (HEARTBEAT) |
| 批量蒸馏 | ✅ /dream | ✅ launchd + kairos_distill.py |
| Append-only 原料层 | ✅ | ✅ raw/ |
| Lint 健康检查 | ✅ | ✅ (蒸馏时) |

## 后台提取提示词（供 Extraction Agent / 子 Agent 使用）

分析对话流，提取"令人惊讶"或"非显而易见"的信息。重点关注：
- 用户纠正 AI 的时刻
- 明确表达的偏好或规则
- 推理得出的隐含规则（需注明 Why）
- 有价值的分析/洞见，值得归档回 MEMORY.md
