# 群聊总结触发条件收紧 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 把 `总结` 触发条件从「群消息正文含『总结』」收紧为「严格 @ 本 bot 且正文含『总结』」，并继续阻断同/低优先级其它插件的响应。

**Architecture:** 在现有 `on_regex` 注册时叠加一条本地自定义 `Rule`，通过遍历 `event.message` 的 `at` 段判断是否指向 `bot.self_id`。`block=True` 保留不变。handler 主体、定时任务、API 调用全部不动。

**Tech Stack:** NoneBot2、`nonebot.adapters.onebot.v11`、`nonebot.rule.Rule`。

---

## File Structure

- Modify: `nonebot_plugin_whats_talk_gemini/__init__.py`
  - 新增导入 `Rule`。
  - 新增内部函数 `_is_at_bot(event, bot)`。
  - 修改 `whats_talk = on_regex(...)` 的 `rule=` 参数。

无需新建文件。仓库无测试套（CLAUDE.md 已说明），通过手动回归验证。

---

### Task 1: 收紧 matcher 的触发规则

**Files:**
- Modify: `nonebot_plugin_whats_talk_gemini/__init__.py:4` (导入)
- Modify: `nonebot_plugin_whats_talk_gemini/__init__.py:46-52` (matcher 注册)

- [ ] **Step 1: 在导入区追加 `Rule`**

把现有第 4 行：

```python
from nonebot import get_bots, get_plugin_config, on_regex, require
```

下方第 9 行：

```python
from nonebot.rule import is_type
```

改为：

```python
from nonebot.rule import Rule, is_type
```

- [ ] **Step 2: 在 matcher 注册之前新增 `_is_at_bot` 函数**

在 `__init__.py` 第 45 行（`# 注册事件响应器` 注释之前）插入：

```python
# 仅当消息中包含指向本 bot 的 at 段时返回 True，
# 不识别"回复 bot"和"昵称前缀"等 to_me() 的宽松形式。
async def _is_at_bot(event: GroupMessageEvent, bot: Bot) -> bool:
    self_id = str(bot.self_id)
    for seg in event.message:
        if seg.type == "at" and str(seg.data.get("qq", "")) == self_id:
            return True
    return False
```

- [ ] **Step 3: 修改 `whats_talk` 的 `rule` 参数**

把现有：

```python
whats_talk = on_regex(
    r"总结",
    priority=5,
    rule=is_type(GroupMessageEvent),
    block=True,
)
```

改为：

```python
whats_talk = on_regex(
    r"总结",
    priority=5,
    rule=is_type(GroupMessageEvent) & Rule(_is_at_bot),
    block=True,
)
```

- [ ] **Step 4: 跑 ruff 校验**

Run: `ruff check . && ruff format --check .`
Expected: 全部 PASS，无 lint 报错。如果 `format --check` 报告需要格式化，运行 `ruff format .` 后再次执行 `ruff format --check .` 直至 PASS。

- [ ] **Step 5: Commit**

```bash
git add nonebot_plugin_whats_talk_gemini/__init__.py
git commit -m "fix: require explicit @bot to trigger 总结 matcher"
```

---

### Task 2: 手动回归验证

仓库没有测试框架，按下表逐项手动测试。每条都要在真实群里跑一遍，并把结果记到本地（不写入仓库）。如有任何一条不符预期，停止后续步骤并回退到 brainstorming 重新评估。

**Files:**
- 不改动文件。

- [ ] **Step 1: 启动 bot**

Run: `nb run`（按 CLAUDE.md，需要 `.env.prod` 中 `DRIVER=~fastapi`）
Expected: bot 正常上线，控制台无 traceback。

- [ ] **Step 2: 验证不带 @ 的消息不再触发**

操作：在白名单群里发送 `总结一下`。
Expected: bot 无任何响应，日志中没有 `handle_whats_talk` 相关条目。

- [ ] **Step 3: 验证 @bot + 总结 触发**

操作：在白名单群里发送 `@bot 总结一下`（真实 at 而不是文字 `@bot`）。
Expected: bot 返回合并转发卡片（标题 `群聊总结`），与改动前一致。

- [ ] **Step 4: 验证「回复 bot」不触发**

操作：在群里回复 bot 任意一条历史消息，回复内容写 `总结`。
Expected: bot 无响应。

- [ ] **Step 5: 验证 @ 其他人不触发**

操作：群里 @ 一个非 bot 成员，正文带 `总结`。
Expected: bot 无响应。

- [ ] **Step 6: 验证 @全体成员不触发**

操作：群里 `@全体成员 总结`（需要群主/管理员权限；若无权限可跳过）。
Expected: bot 无响应。如果跳过，记录"未验证"。

- [ ] **Step 7: 验证私聊不触发**

操作：私聊 bot 发送 `总结`。
Expected: bot 无响应（已有 `is_type(GroupMessageEvent)` 兜底）。

- [ ] **Step 8: 验证定时推送仍正常**

可选：临时把 `wt_push_cron` 调到 1-2 分钟后，重启 bot 等待触发；或者等下一次原定 cron。
Expected: 定时任务到点正常生成并推送总结。如不便临时改 cron，可只确认日志中 apscheduler 已注册 `push_whats_talk` 任务。

---

## Self-Review

- **Spec coverage** — spec 三项目标（严格 @、保留 block=True、不影响定时推送）都对应到 Task 1 的 step 3（rule 叠加 + 保留 block）和 Task 2 的 step 8（定时推送验证）。spec 行为对照表中的 7 个场景对应 Task 2 step 2–7（@全体成员视权限可能跳过，已注明）。
- **Placeholder scan** — 没有 TBD / TODO / "类似上面"。每个改动步骤都给出了具体代码或具体命令。
- **Type consistency** — `_is_at_bot` 在所有任务里命名一致；`Rule` 导入与用法一致；`bot.self_id` 强制 `str()` 与 segment `qq` 字段比较的处理统一。

---

Plan complete and saved to `docs/superpowers/plans/2026-05-13-at-only-trigger.md`. Two execution options:

1. **Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.
2. **Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
