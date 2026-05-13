# 群聊总结触发条件收紧设计

日期：2026-05-13
作者：brainstorming 会话
范围：`nonebot_plugin_whats_talk_gemini/__init__.py` 的消息匹配器

## 背景

当前插件通过 `on_regex(r"总结", priority=5, rule=is_type(GroupMessageEvent), block=True)` 注册响应器。由于正则仅匹配"总结"二字，任何含"总结"的群消息都会进入 handler，触发一次完整的 LLM 调用。用户希望：

1. 仅在消息里真实 `@ bot` 时才触发。
2. 继续拦截事件，避免同优先级或低优先级的其它插件也对同一条消息响应。

## 目标

- 收紧触发条件为「消息中存在指向本 bot 的 at segment 且正文含『总结』」。
- 保留既有的拦截能力（`block=True`）。
- 不影响 `push_whats_talk` 定时推送逻辑。
- 不改变 handler 主体、API 调用、昵称压缩、输出格式等下游流程。

## 非目标

- 不支持"回复 bot"或"以 bot 昵称开头"等 `to_me()` 的宽松识别方式。用户明确要求仅严格 @。
- 不修改配置文件、不引入新的配置项。
- 不调整定时推送 cron、群列表、API key 失败切换等。

## 设计

### 方案选型

评估过三种方案，最终采用方案 A：

- **A（采用）**：保留 `on_regex`，在 rule 里叠加一个自定义 checker，仅当消息段中出现 `at` 且目标 QQ 等于 `bot.self_id` 时放行。
- **B（否决）**：改用 `on_command` 并设置 `COMMAND_START=["@"]`，会影响整个 bot 的命令前缀，副作用过大。
- **C（否决）**：使用 `to_me()` 再手动排除回复和昵称前缀分支，写法绕，不如直接写严格的 at 判定。

### 代码改动

文件：`nonebot_plugin_whats_talk_gemini/__init__.py`

1. 新增本地 rule checker：

   ```python
   from nonebot.rule import Rule

   async def _is_at_bot(event: GroupMessageEvent, bot: Bot) -> bool:
       self_id = str(bot.self_id)
       for seg in event.message:
           if seg.type == "at" and str(seg.data.get("qq", "")) == self_id:
               return True
       return False
   ```

   - 只检查直接的 `at` 段；不处理 reply、不处理昵称前缀。
   - `qq` 统一转字符串比较，避免 int/str 类型差异。
   - `@全体成员` 的 `qq` 为字符串 `"all"`，自然不会命中。

2. 修改 matcher 注册：

   ```python
   whats_talk = on_regex(
       r"总结",
       priority=5,
       rule=is_type(GroupMessageEvent) & Rule(_is_at_bot),
       block=True,
   )
   ```

   - `block=True` 维持不变，继续拦截同/低优先级其它 matcher。
   - 正则保持 `r"总结"`，正文中任何位置含"总结"即可。

### 不变的部分

- `handle_whats_talk` 整个 handler 主体。
- `get_history_chat` / `chat_with_gemini` / `get_group_member` 等所有辅助函数。
- `push_whats_talk` 定时任务——它不走 matcher，不受新规则影响。
- 插件元数据 `__plugin_meta__`（`usage` 描述可在后续 PR 里微调，但本次不动，避免扩大范围）。

## 行为对照

| 场景 | 当前 | 改动后 |
|---|---|---|
| 群消息「总结一下」（无 at） | 触发 | 不触发 |
| 群消息「@bot 总结一下」 | 触发 | 触发 |
| 回复 bot 一条消息，内容含「总结」 | 触发 | 不触发 |
| 群消息「@其他人 总结一下」 | 触发 | 不触发 |
| 群消息「@全体成员 总结」 | 触发 | 不触发 |
| 私聊 bot「总结」 | 不触发（已有 `is_type(GroupMessageEvent)`） | 不触发 |
| 定时推送 | 正常 | 正常 |

## 测试

仓库没有测试框架。靠手动回归覆盖上表中的关键场景：

1. 群里发 "总结一下" — 预期无响应。
2. 群里发 "@bot 总结" — 预期返回合并转发总结。
3. 回复 bot 一条历史消息内容为 "总结" — 预期无响应。
4. 群里发 "@其他人 总结" — 预期无响应。
5. 等一次定时推送周期（或临时调 cron）确认定时任务仍正常。

## 风险与回滚

- 风险低：只改 matcher 规则，handler 和定时任务不变。
- 回滚：删除 `Rule(_is_at_bot)` 叠加和 `_is_at_bot` 定义即可恢复原行为。

## 验收

- `ruff check .` 和 `ruff format .` 通过。
- 手动回归上面的 5 个场景结果符合预期。
