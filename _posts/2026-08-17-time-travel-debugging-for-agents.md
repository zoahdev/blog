---
layout: post
title: "Time-travel debugging for agents: replaying a session from its ground truth"
date: 2026-08-17
---

# Time-travel debugging for agents

**Summary.** Debugging an agent by scrolling the terminal only shows the last
N lines. DeepSeek Harness already records everything — every reasoning delta,
every tool call, every result, with seq + timestamps — but in a
concatenated-zstd, packed-row format nobody can read by hand.
[dsh-replay](https://github.com/zoahdev/dsh-replay) decodes that ground-truth
log and renders it as a readable timeline: replay, diff, and summarize an
entire agent trajectory from `session.jsonl.zstd`.

---

## Why ground truth matters

Screenshots and terminal scrollback are not a debugger. The session log is:

```text
session.jsonl.zstd — every event, in order, with seq + timestamps
```

The problem is the format: concatenated Zstandard frames wrapping packed
`text-chunks` / `reasoning-chunks` / `tool-call-chunks` rows. It is the exact
`@deepseek-ai/dsh-session` wire format, and it is not human-readable.

## What dsh-replay does

```sh
# decode and render a full timeline
npx dsh-replay <session-id> --out replay.html

# point at a file directly
node bin/replay.mjs --file ~/.dsh/sessions/.../session.jsonl.zstd

# per-tool call counts, error count, latency as JSON
node bin/replay.mjs <session-id> --stats

# where did two runs diverge?
node bin/replay.mjs diff <id-a> <id-b> --diff-html
```

Under the hood:

1. **Decode** the concatenated-Zstandard container and the packed rows (a
   zero-dependency re-implementation of the upstream wire format);
2. **Reconstruct** turns → steps → user messages → reasoning → assistant text
   → tool calls with results (success/error);
3. **Render** a self-contained dark-mode HTML timeline;
4. **Diff** two sessions turn-by-turn — the divergence point is the bug;
5. **Summarize** per-tool counts, errors, and latency as machine-readable JSON.

No `@deepseek-ai/dsh` dependency; just Node ≥ 22.19 with its bundled zstd.

## Correctness is the feature

A decoder that "mostly works" is worse than none — it produces confident
misreadings. The decoder mirrors the upstream `scanZstdFrames` +
`decodeStorageRecord` format, and tests cover packed-row expansion, tool-call /
result reconstruction, and the diff summarizer.

This is the same discipline as the rest of the ecosystem work: the tool is
only worth shipping if it can be proven against the actual format.

---

## 中文摘要

调试 agent 最常用的方式是翻终端，但只能看到最后几行。DSH 的会话日志其实记录
了一切（推理、工具调用、结果、seq + 时间戳），只是压缩在 zstd 打包格式里没法
人读。dsh-replay 零依赖解码这个真实线格式，重建 turn → step → 工具调用全轨迹，
渲染成深色 HTML 时间线，还能 diff 两次运行（分歧点就是 bug）、输出 JSON 统计。
解码器逐字段对齐上游 scanZstdFrames + decodeStorageRecord，测试覆盖打包行展开
和工具结果重建——"基本能用"的解码器比没有更糟，对得上真实格式才值得发布。
