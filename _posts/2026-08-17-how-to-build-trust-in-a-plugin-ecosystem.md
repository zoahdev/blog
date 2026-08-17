---
layout: post
title: "How to build trust in a plugin ecosystem"
date: 2026-08-17
---

# How to build trust in a plugin ecosystem

**Summary.** Plugin ecosystems die from distrust, not from lack of ideas:
supply-chain attacks, broken installs, and unverifiable quality claims chase
users away. Over two weeks the DeepSeek Harness community built a trust layer
out of four pieces — verification discipline, poison detection, registry
contracts, and open diagnostics. This post is the map of that layer and the
rules that keep it honest.

---

## The problem

An ecosystem with 900+ plugins is a discovery win and a trust liability at
the same time. Users need to answer three questions before they install
anything:

1. **Will it load?** (dependency compatibility, build, manifest)
2. **Is it malicious?** (obfuscated exfiltration, hidden shell launches)
3. **Can I verify the claim?** (what does "works" even mean, who checked)

If any of the three is answered by vibes, the ecosystem loses users.

## Layer 1: verification discipline

`dsh-plugin-doctor` turns "will it load" into a machine-checkable question:
manifest → build → pack → fresh profile install → boot → **real tool
invocation**. The CI template does not stop at "plugin loads"; it calls the
tool's handler and asserts the result. Exit codes are stable (0 pass / 1
fixable / 2 not a plugin), output is JSON, and the profile-shadow check
catches the #1697 dual-instance trap before users hit it.

## Layer 2: poison detection

`dsh-poison-guard` scans plugins before install with AST parsing (js-x-ray)
plus anti-obfuscation decoding, flagging hidden exfiltration, `eval`-of-dynamic-
content, and concealed shell invocations. Its scope is stated honestly: a
heuristic review aid, not a sandbox. The sandbox question is an OS boundary
problem (#1863/#1923), and pretending a static scan is a security boundary
would be exactly the kind of claim that destroys trust.

## Layer 3: registry contracts

`dsh-subscribe` is a Steam-style marketplace: browse 900+ plugins, subscribe
in the browser, one command installs everything into the profile. The registry
entry format is a contract (RFC #1846, community registry contract v2):

- `id` stable, owner-suffixed for duplicates;
- `install.spec` is authoritative, never guessed from the homepage;
- `verified: true` means "curator exercised CI + release + install + runtime
  smoke" — **not** "security audited";
- `source` preserves provenance for mirrored entries.

The narrow definition of `verified` is the point: a wide claim would be
useless.

## Layer 4: open diagnostics

Five independent tools converged on one `dsh-doctor/v1` envelope (schema,
lowercase status, checks with names/details), plus a check-lifecycle draft
where "checks are data". The vocabulary freeze candidate means CI scripts can
match `checks[].name` across implementations. Diagnostics that admit their
gaps (sandbox denials, approval policy — no offline probe can see them) are
the ones you can build on.

## The rules that keep it honest

1. **Narrow claims beat impressive ones.** "verified" never means "audited";
   "review aid" never means "sandbox".
2. **Rejections are part of the product.** Bitbucket was excluded from the
   catalog because its "public" API returns 404/410 anonymously — the rejection
   is documented in the README.
3. **Everything is reproducible.** Patches carry root causes + regression
   tests; the weekly map's numbers come from live sources; the ledger
   separates "ready to submit" from "diagnosed, no patch yet".
4. **Contracts over chaos.** One envelope, one vocabulary, one registry
   schema — agreed by shipping reference implementations, not by committee.

The ecosystem map (weekly editions) and the patch ledger live in
[dsh-ecosystem](https://github.com/zoahdev/dsh-ecosystem) and
[dsh-docs](https://github.com/zoahdev/dsh-docs).

---

## 中文摘要

插件生态死于不信任，而不是缺想法。DSH 社区两周内搭了四层信任设施：
① 验证纪律（doctor：manifest→build→pack→新 profile 安装→启动→真实调用 tool，
退出码稳定、输出 JSON）；② 投毒检测（poison-guard：AST + 反混淆，明说是审查
辅助不是沙箱）；③ 注册表契约（subscribe：900+ 插件一键订阅，
`verified` 只表示"策展人跑过 CI+安装+运行时冒烟"，绝不等同安全审计）；
④ 开放诊断（5 个工具收敛到 dsh-doctor/v1，检查名词表冻结候选，敢承认盲区）。
四条规则：窄而准的声明 > 夸张的声明；拒绝记录也是产品；一切可复现；
用契约而不是争论达成一致。
