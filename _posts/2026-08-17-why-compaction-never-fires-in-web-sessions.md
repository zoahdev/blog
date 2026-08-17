---
layout: post
title: "Why compaction never fires in DSH web sessions"
date: 2026-08-17
---

# Why compaction never fires in DSH web sessions

**Summary.** Long DeepSeek Harness web sessions hit provider
context-exceeded errors even though "compaction should handle this". Two
independent root causes, both verified against the installed
`0.1.0-rc.6` bundle and against runtime traces:

1. The web profile **ships compaction disabled** — the web-app patch layer
   sets `disabled: true` on the host-plane copies of `compaction-basic`,
   `command-compact`, and `tool-result-pruner`, and the standard preset's
   own realm copy never activates for web sessions.
2. Even when re-enabled, compaction **still does not fire** because the token
   meter's fixed estimate (4 chars/token, flat) undercounts dense tool-schema
   and tool-result surfaces, so the pressure threshold is never reached before
   the provider rejects.

Thread: [discussion #2107](https://github.com/deepseek-ai/deepseek-harness/discussions/2107).

---

## What the composition actually contains

Verified against the installed bundle, not repo-HEAD:

```text
node_modules/@deepseek-ai/dsh/node_modules/@deepseek-ai/dsh-base/cordis.patch.yml
  line 284: - id: compaction-basic
  line 289: - id: command-compact
  line 360: - id: tool-result-pruner
```

These mount the three compaction rows on the **host** plane. The standard
agent preset mounts the same three again inside its own realm:

```yaml
# apps/cli/config/agent-presets/standard/agent.cordis.yml
isolate:
  compaction: true
  toolResultPruner: true
```

And `packages/bundle/web-app/cordis.patch.yml` then sets `disabled: true` on
the three host-plane rows. Result: the host copy is deliberately off, and the
preset copy is supposed to be the supplier for web sessions.

## Runtime finding: the preset copy never activates

Re-enabling the host copy is composition-verified (`dsh --profile web
--dump-config` shows `disabled: false` and the `thresholdRatio` override),
but a 22-step file-read-heavy web session produced **821 session events and
zero `compaction/start` or `compaction/summary` events**. The provider
rejected with `Context size has been exceeded` while the meter showed no
pressure. Sweep of every `session.jsonl.zstd` on the install: zero
compaction-ish events in any profile.

That points to the preset-realm backend never joining the session, not to the
host copy's config.

## The second root cause: the estimator undercounts

`packages/llm/token-meter/src/estimate.ts` prices everything at a flat
`CHARS_PER_TOKEN = 4` — text, reasoning, tool-call arguments, tool results,
system prompt, and tool schemas:

```ts
const CHARS_PER_TOKEN = 4
```

For a session with 47 tool schemas (47,229 chars) and repeated large file
reads, real provider tokenization is far denser than 4 chars/token. The meter
therefore reports "no pressure" while the provider is already at its limit.
The compaction threshold (default 0.8 of the window) is never reached in
time — the undercount makes the gate arrive late or never.

## What this means for users

- On `0.1.0-rc.6`, **tuning `thresholdRatio` alone will not fix web
  sessions**: compaction is disabled by default, and even enabled it does not
  fire because of the meter.
- Reliable mitigations: start a new session with conclusions carried over;
  set an accurate `contextWindow` for the model (unknown models default to
  1,000,000 tokens, which can be far above a local model's real window); trim
  the output budget.
- Track upstream: #2107. Both diagnoses are recorded in the patch ledger's
  "tracked diagnoses" section (D1 meter estimate, D2 web compaction realm)
  until there is a verified cherry-pick-ready fix.

---

## 中文摘要

长会话在 DSH web 里撞上下文上限，根因有两层：① rc.6 的 web profile 默认
把 compaction 三个插件禁用（web-app 补丁层 `disabled: true`，标准预设自己的
realm 副本对 web 会话不生效）；② 即使手动启用，token meter 用固定的
4 chars/token 估算，严重低估密集工具 schema 和工具结果，压力阈值永远到不了，
provider 先拒绝。所以"调 thresholdRatio"在 rc.6 上基本无效；可靠做法是
新开会话 + 配置准确的 contextWindow + 压输出预算。跟踪帖 #2107，诊断已记入
补丁台账 D1/D2。
