# OpenClaw MindPalace 自检手册 (Self-Check Guide)

为了确保本 Skill 在你的 OpenClaw 环境下生效，请按照以下步骤进行手动和自动校准。

## 第一步：环境验证 (Infrastructure Check)
在终端运行：
1. `which rg`: 确保系统中已安装 ripgrep。
2. `ls -R .openclaw/memory/`: 确保至少存在 `INDEX.md` 和 `topics/` 目录。
3. `openclaw skills list`: 确认 `memory_system` 在已加载列表。

## 第二步：功能性冒烟测试 (Smoke Test)
向 OpenClaw 发送以下测试指令，观察其反应：

### 测试 1：主动记忆保存
**指令**：`"记一下，我是一个喜欢用 Bun 而不是 npm 的前端专家。"`
**预期行为**：
1. AI 调用 `save_memory` 相关工具（或手动执行 file 写操作）。
2. `.openclaw/memory/topics/` 下生成了 `user_preferences.md`。
3. `INDEX.md` 中新增了一行关于 `Bun` 的摘要。

### 测试 2：跨会话召回 (Recall)
**指令**（在另一个新会话中）：`"我之前提到的 Node 运行时偏好是什么？"`
**预期行为**：
1. AI 提示它正在查阅记忆索引。
2. 回复：“你提过你是前端专家，且明确偏好使用 Bun 而非 npm。”

### 测试 3：判别逻辑验证 (Refusal Test)
**指令**：`"把这个项目的 package.json 里的依赖列表记在记忆里。"`
**预期行为**：
1. **拒绝**。AI 应该回复类似：“根据记忆准则，依赖列表是可推导的代码事实，不应占用持久化记忆空间。”

## 第三步：常见隐患与注意事项
1. **路径冲突**: 确保 OpenClaw 的运行账户对 `.openclaw/memory/` 有 **R/W 权限**。
2. **索引溢出**: `INDEX.md` 如果超过 200 行，将导致每个 Prompt 增加数千 Token 的负担。请定期手动 review 索引。
3. **并发幻觉**: 如果你在后台“梦境”蒸馏时手动修改记忆文件，可能会导致文件被 AI 覆盖。建议在非活跃时间进行大规模整理。
4. **验证必选**: 永远不要相信一年前的记忆。如果记忆说 `utils.js` 在 A 目录，但在推荐前请先 `ls` 一下。
