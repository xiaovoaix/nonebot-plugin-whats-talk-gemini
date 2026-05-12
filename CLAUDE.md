# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

`nonebot-plugin-whats-talk-gemini` — a NoneBot2 plugin (OneBot v11 adapter) that summarises recent group chat via an LLM. Two entry points: on-demand via any group message matching the regex `总结` (e.g. `群聊总结`, `总结一下`), and scheduled push via apscheduler cron. Summaries are delivered as OneBot *forward-message nodes* (`send_group_forward_msg`), not plain text, so QQ clients render them as a merged card.

## Commands

```bash
# Lint (config in pyproject.toml — line-length 120, ruleset F/E/W/I/UP/RUF)
ruff check .
ruff format .

# Build wheel + sdist locally (CI publishes on tag push via .github/workflows/pypi-publish.yml)
python -m build --sdist --wheel --outdir dist/ .
```

The plugin is loaded inside a host NoneBot2 project, not run standalone. There is no test suite.

## Architecture

Everything lives in `nonebot_plugin_whats_talk_gemini/__init__.py` (~350 lines) plus `config.py`. Flow:

1. `get_history_chat` calls OneBot's `get_group_msg_history` and keeps only `text` segments (drops CQ codes, images, etc.). Records the timestamp of the first and last kept message.
2. `chat_with_gemini` does two things before the HTTP call:
   - **Nickname compression** — replaces each distinct sender name with a 3-hex code (`000`..`FFF`, so ~4096 users max) to shrink prompt length. After the model responds, codes are substituted back **in descending length order** to avoid a short code matching a prefix of a longer one. Any code edit here must preserve that ordering.
   - **Dual API shape** — branches on `"googleapis.com" in wt_base_url`:
     - Google native: `POST {base_url}/models/{model}:generateContent?key=...`, body shape `{"contents": [{"parts": [{"text": ...}], "role": "user"}, ...]}`, no `Authorization` header.
     - OpenAI-compatible: `POST {base_url}/chat/completions`, standard `messages` + `Bearer` auth. Only this branch forwards `wt_thinking` (as `reasoning_effort: "high"` + `extra_body.thinking.type`).
   - Response parsing accepts both `choices[0].message.content` and `candidates[0].content.parts[*].text`.
3. API key failover — iterates `wt_ai_keys`; a `429` on one key falls through to the next, any other HTTP error aborts.
4. Output — always shipped as a single forward-message node via `bot.send_group_forward_msg`. Changing this to a plain message will change how QQ renders it.

Scheduled pushes reuse the same two helpers. `wt_push_cron` is parsed by a local `parse_cron_expression` (strict 5-field format) and registered on module import via `@scheduler.scheduled_job`, so changing the cron value requires a restart.

## Config notes

See `config.py` for the full schema. Points that aren't obvious from the README:

- `wt_ai_keys` is validated at import time — an empty list raises `ValueError` and the plugin fails to load.
- `wt_thinking` is tri-state: `None` means "don't send the field at all", not "off". `False` explicitly sends `thinking.type: "disabled"`.
- `wt_base_url` selects API dialect by substring match on `googleapis.com`. Any OpenAI-compatible proxy (even one proxying Gemini) goes through the OpenAI branch.

## Prompt

The prompt is hard-coded inside `chat_with_gemini` and enforces a specific output layout (`【采集时间】`, `【整体总结】`, `【话题N】`, `【互动评价】`) with a no-markdown rule. The prompt embeds the full `code = nickname` table so the model can still refer to users meaningfully; the post-processing substitution is a safety net for when the model echoes codes verbatim.
