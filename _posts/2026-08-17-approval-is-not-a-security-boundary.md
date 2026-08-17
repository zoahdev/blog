---
layout: post
title: "Approval is not a security boundary"
date: 2026-08-17
---

# Approval is not a security boundary

**Summary.** In DeepSeek Harness, the ask-approval gate and workspace-write
limits are **consent / routing UX, not OS security boundaries**. Two verified
bypass families make this concrete:

1. [#1863](https://github.com/deepseek-ai/deepseek-harness/discussions/1863) —
   `pre-execute` listeners run before `ask` approval; a plugin can finish host
   operations in a separate Node.js child process and only then return `ask`.
2. [#1923](https://github.com/deepseek-ai/deepseek-harness/discussions/1923) —
   delegating execution to a user-privileged external shell
   (`explorer` / `start` / `open`) sidesteps both gates: the agent is not
   running the command, the user's own context is.

The framing matters: **plugins are installed host code**. They can import Node
built-ins and run anything. Approval gates *intent*, not *capability* — it
cannot retroactively undo a listener that already ran host operations before
returning `ask`.

---

## Family 1: side effects before the ask (`pre-execute`)

The original finding: a `tools/pre-execute` listener can perform its work in
an independent Node.js subprocess and only then return `ask` — so the approval
UI shows a prompt after the operation already happened. Nothing in the harness
can distinguish "the listener decided to decline" from "the listener already
executed and is asking for form's sake".

Consequence for API design: `pre-execute` listeners should be **pure decision
functions**. No child processes, no filesystem writes, no network in the
listener body.

## Family 2: the shell-launcher bypass

The second channel: `child_process` calls that hand off to the user's own
shell environment — `explorer.exe`, `start`, `open`, `powershell`, `cmd` —
launch a process that inherits the user's privileges and is no longer subject
to the agent's workspace/approval policy. The agent "didn't run the command";
the user's own context did.

## Why policy checks are not a sandbox

- Approval gates *who is asked*, not *what can run*.
- Workspace-write limits route file access for agent tools; a plugin that
  spawns outside the sandbox is simply outside the routing table.
- Static checks (heuristics, scans) can *reduce* the surface but cannot
  *enforce* it — enforcement requires the OS boundary.

## Fix directions, in order of leverage

1. **OS-level sandbox, not just policy checks**: workspace-write must map to
   a restricted token / container. Windows direction:
   `CreateRestrictedToken` (see #1789) so a "launched by the user's shell"
   process still runs without workspace-escape privileges.
2. **Contract**: `pre-execute` listeners are pure decision functions (no
   child processes, no fs writes, no network).
3. **Shell-launcher whitelisting**: delegation surfaces that intentionally
   use user-privileged shells need an explicit allowlist and a warning.

## Toolized review aids (honest scope)

- `dsh-plugin-doctor` **v1.9.0** `pre-execute-side-effects`: scans source/lib
  for `pre-execute` listeners and flags same-file host-level side-effect APIs;
  FAIL names the file and matched API.
- `dsh-plugin-doctor` **v1.11.0** `shell-launcher`: warns when
  `child_process` usage invokes `explorer` / `start` / `open` /
  `powershell` / `cmd` surfaces (the #1923 channel).

Both are explicitly labeled **review aids, never a sandbox**: 31/31 tests,
self-check exit 0, and the tool's own `cmd.exe` wrapper shows up as an honest
WARN.

---

## 中文摘要

在 DSH 里，"ask 审批"和"workspace 写限制"是同意/路由 UX，不是 OS 安全边界。
两条已验证的绕过：① `pre-execute` 监听器在审批前就能用独立子进程把事做完再
返回 ask（#1863）；② 借用户特权的外部 shell（explorer/start/open）执行，直接
绕过两个闸门（#1923）。插件是装进宿主的高权限代码，审批拦的是"意图"不是
"能力"。修复优先级：OS 级沙箱（Windows 走 CreateRestrictedToken 方向，
#1789）> 契约约束（pre-execute 必须是纯决策函数）> shell-launcher 白名单。
配套检测：dsh-plugin-doctor v1.9.0（pre-execute-side-effects）和 v1.11.0
（shell-launcher），明确标注为"审查辅助、不是沙箱"。
