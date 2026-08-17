---
layout: post
title: "Five implementations, one contract: how the DSH doctor ecosystem converged"
date: 2026-08-17
---

# Five implementations, one contract

**Summary.** In one week, five independent community tools for diagnosing
DeepSeek Harness converged on a single machine-readable contract
(`dsh-doctor/v1`) — not by committee, but by publishing specs with running
reference implementations and letting the format win on evidence. Thread:
[#1719](https://github.com/deepseek-ai/deepseek-harness/discussions/1719).

---

## The starting point: five tools, five formats

The community shipped "dsh doctor" five times:

| Tool | Author | Shape |
|---|---|---|
| dsh-plugin-doctor | zoahdev | author-side preflight + profile tripwire |
| dsh-doctor | moonquake2004 | 19 offline checks, zero deps |
| dsh-doctor | ciceroyang | zero-dep single-file CLI + log health |
| dsh-win32 doctor | sjh9714 | win32-scoped, ships with the platform |
| dsh-diagnose | worm-ai | symptom → mechanism → check skill |

Same niche, five output shapes. Nobody could pipe one tool into another, and
CI couldn't consume any of them uniformly. That was the actual problem — not
the tools themselves.

## What made convergence possible

1. **A published envelope spec.** `{ schema: "dsh-doctor/v1", generatedAt,
   profile, exitCode, summary, ok, checks: [{name, status, detail}] }` with
   lowercase status (`pass` / `warn` / `fail`) and a minimal compatible subset
   of `{ok, checks}`.
2. **Reference implementations, not promises.** Each author shipped the
   envelope in a real release within the same day: zoahdev v1.6.0,
   moonquake2004 v0.2.4, ciceroyang v0.3.0, sjh9714 v0.8.x, worm-ai
   `--doctor-json` (generation 61+).
3. **Independent verification with honest gap marks.** moonquake2004 mapped
   all 16 of dsh-diagnose's symptom families to its 28 checks with ✅ direct /
   ⚠️ partial / ❌ gap — including two honest gaps (sandbox denials, approval
   policy) that no offline probe can see.
4. **Provenance without a global registry.** The `tool` field was added as
   an optional provenance marker instead of forcing a global check-id
   registry early — "checks are data" (declarative, read-only probes) per the
   check-lifecycle draft.

## The vocabulary freeze candidate

The check-name vocabulary went through four drafts (r1 → r4), incorporating:

- the `skip` status with a mandatory reason (sjh9714);
- corrected `node` provenance — repo-declared `engines`, not folklore
  (ciceroyang, escalated to #2259);
- `pass` → `ok` normalization debate, settled at the freeze boundary.

The r4 draft is a freeze candidate: same check names, interchangeable across
implementations, CI scripts matching only `checks[].name` + `status`.

## Why this is a repeatable pattern

The convergence did not happen because anyone "won". It happened because each
round published:

1. a spec diff,
2. a running implementation of that diff,
3. independent verification against the actual source,
4. an honest list of what the tool still cannot see.

The last point is the one that builds trust: a diagnostic that admits its
gaps is one you can build on.

---

## 中文摘要

一周内五个社区工具（dsh-plugin-doctor / moonquake2004 dsh-doctor /
ciceroyang dsh-doctor / dsh-win32 / dsh-diagnose）收敛到同一个
`dsh-doctor/v1` 契约，靠的不是开会，而是"规格 + 当天就上线参考实现 +
独立核对 + 诚实标注盲区"四件套。检查名词表走到 r4 冻结候选，`tool` 字段
解决来源溯源而不抢注册表。经验：能承认自己看不到什么的诊断工具，才值得
别人在此基础上继续盖楼。完整讨论：#1719。
