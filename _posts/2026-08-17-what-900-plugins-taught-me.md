---
layout: post
title: "What 900 plugins taught me about an ecosystem's first week"
date: 2026-08-17
---

# What 900 plugins taught me about an ecosystem's first week

**Summary.** DeepSeek Harness's plugin registry crossed **908 entries** within
its first week of tracking — up from ~554. This post reads that data the way
an ecosystem map-maker should: what the category distribution says, what the
**verification gap** quantifies, and what happens next. All numbers are live
registry data, not estimates.

---

## Growth

```text
Week 1 registry:  ~554
Current registry:  908   (+~64%)
```

For a developer-preview harness whose upstream PR channel is closed, that is
not a hype curve — it is people building against the current API while the
API is still moving. Every one of those 908 entries is a bet on the platform.

## Category distribution

| Category | Count | Share |
|---|---|---|
| tools | 208 | 23% |
| ui | 208 | 23% |
| dev | 108 | 12% |
| workflow | 66 | 7% |
| notify | 59 | 6% |
| session | 58 | 6% |
| memory | 48 | 5% |
| fun | 36 | 4% |
| theme | 35 | 4% |
| skill | 32 | 4% |
| model | 30 | 3% |
| market | 19 | 2% |

Two readings:

1. **The platform is past "experiment".** tools + ui = 46% of the ecosystem.
   The community is shipping capability, not just demos.
2. **The agent-native categories are forming.** memory (48), workflow (66),
   session (58), skill (32) — these are not ported tools, they are
   harness-shaped primitives. That is the sign of a platform, not a wrapper.

fun (36) and theme (35) are the health signal: an ecosystem that people
personalize is an ecosystem people stay in.

## The verification gap

```text
verified: 21 / 908  (2.3%)
```

This is the number that matters. Discovery is solved — 908 entries, browseable
and searchable. **Trust is not**: 2.3% of the registry carries the narrow
"curator exercised CI + install + runtime smoke" claim. The gap is not a
criticism of the 887 unverified entries; it is the quantified reason the
doctor / poison-guard / registry-contract work exists. An ecosystem with
900 plugins and no verification layer is a discovery win and a trust
liability at the same time.

## What happens next

- The verified fraction needs to climb, and the `verified` claim must stay
  narrow (CI + install + runtime smoke, never "audited").
- The agent-native categories (memory, workflow, session) will keep growing —
  they are where the harness differs from a CLI wrapper.
- Registry interop (one contract, RFC #1846) determines whether 908 entries
  in one storefront become 908 entries in every storefront.

The weekly map continues at [dsh-ecosystem](https://github.com/zoahdev/dsh-ecosystem),
with every number traceable to a live source.

---

## 中文摘要

DSH 插件注册表第一周从约 554 涨到 908（+64%）。分类分布说明平台已经过了
"实验期"：tools+ui 占 46%，社区在交付能力而不只是 demo；memory/workflow/
session/skill 这些 agent 原生类别正在成型——这是平台的信号不是包装器的信号。
最关键的数是验证缺口：**908 个里只有 21 个 verified（2.3%）**。发现的问题解决
了，信任没有——这正是 doctor/投毒扫描/注册表契约存在的量化理由。下一步：
verified 比例要涨且定义保持窄、agent 原生类别继续长、注册表互操作（#1846）
决定 908 个条目是"一个商店"还是"所有商店"。
