---
layout: post
title: "The symbol that broke every plugin in DeepSeek Harness"
date: 2026-08-17
---

# The symbol that broke every plugin

**Summary.** Installing a plugin that depends on `@deepseek-ai/dsh-tools` broke
every subsequent tool call with `Cannot read properties of undefined (reading
'prepare')`. Root cause: the tool runtime scheduler is keyed on a
module-local `unique symbol`, and a plugin install can pull a second physical
copy of `dsh-tools` into the same process — two copies, two symbols, one
missing scheduler. The fix is not just `Symbol.for`; it is a shared registry
key **plus** a protocol-version guard so version skew fails loudly instead of
silently. Full thread: [discussion #1697](https://github.com/deepseek-ai/deepseek-harness/discussions/1697).

---

## The symptom

On `0.1.0-rc.6`:

```text
Cannot read properties of undefined (reading 'prepare')
```

...for **every** tool call, in a session that worked before a plugin install.
Pure model requests still worked; any actual tool execution crashed. Users
reported it as "session completely non-diagnosable".

## The root cause

The scheduler service is registered under a symbol that is local to each copy
of the package:

```ts
// packages/core/tools/src/index.ts
export const TOOL_RUNTIME_SCHEDULER: unique symbol = Symbol('@deepseek-ai/dsh-tools.scheduler')
```

A plugin that depends on `dsh-tools` can install a **second physical copy**
into the profile (same version, different module instance). Now the process
has two `Symbol('@deepseek-ai/dsh-tools.scheduler')` values — and they are
**not equal**. The tool runtime resolves its provider through the symbol;
when the two sides use different instances, the lookup returns `undefined`,
and the call site throws on `.prepare`.

This is why the failure was "completely non-diagnosable" from logs: nothing in
the error names the package, the version, or the duplicate.

## Mechanism-level verification

Two physical copies of `dsh-tools@0.1.0-rc.6` in one process:

| Setup | Result |
|---|---|
| One copy | ✅ tools work |
| Two copies, same version | ❌ `undefined.prepare` |
| Two copies, version skew (rc.5 + rc.6) | ❌ same crash family |

The three-state matrix is the whole story: it is a **module-identity** bug, not
a version-only bug.

## Why not just `Symbol.for`

`Symbol.for(key)` collapses module identity into the global registry. It fixes
the crash — but it also **silently** fuses two different package versions.
rc.5's scheduler and rc.6's scheduler would resolve to the same key, and the
first one to register wins. A version-skew install stops crashing and starts
misbehaving in ways that are much harder to trace.

The upgraded fix, staged on
[`fix/tool-runtime-scheduler-symbol-for`](https://github.com/zoahdev/deepseek-harness/tree/fix/tool-runtime-scheduler-symbol-for):

1. `Symbol.for` shared registry key (fixes the duplicate-instance crash);
2. `TOOL_RUNTIME_SCHEDULER_PROTOCOL_VERSION` guard (turns version skew into a
   loud, actionable error instead of a silent mismatch).

## The tripwire

The same failure family also lives in the profile, not just the bundle:
`dsh-plugin-doctor --profile <name>` runs a `profile-shadow` check that FAILs
when a profile pins an older `@deepseek-ai` core package over the host's. A
clean profile passes; a shadowed one fails with the exact package name and
path. That turns "completely non-diagnosable" into a one-command answer.

## Where this sits now

- Fix: fork branch, mechanism-verified, cherry-pick ready (patch queue #1 in
  [RFC #2486](https://github.com/deepseek-ai/deepseek-harness/discussions/2486)).
- Diagnosis: `dsh-plugin-doctor --profile` (author-side + profile tripwire).
- Content rescue for sessions that already got stuck: `dsh-shelf rescue`.

---

## 中文摘要

一个模块内的 `unique symbol` 让 DeepSeek Harness 在安装插件后所有工具调用
崩溃（`undefined.prepare`）：插件带来第二份 `dsh-tools`，同一个进程里出现
两个不同的 Symbol，调度器查不到。修复不是简单换成 `Symbol.for`——那会把
崩溃变成更隐蔽的静默版本错配；正确做法是共享注册表键 + 协议版本号守卫，
让版本不一致大声报错。配套的 `dsh-plugin-doctor --profile` 的
`profile-shadow` 检查可以一条命令定位宿主遮蔽。完整讨论：
[#1697](https://github.com/deepseek-ai/deepseek-harness/discussions/1697)。
