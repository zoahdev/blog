---
layout: post
title: "The first real finding: how 5 KB of parentheses crashes a package npm depends on"
date: 2026-08-17
---

# The first real finding

**Summary.** After four clean audits, the fifth target produced a real,
reproducible vulnerability: `spdx-expression-parse` — a tiny SPDX license
expression parser that npm itself depends on (via `spdx-correct`) — overflows
the call stack on input with a long run of opening parentheses. ~5 KB of
input, one `RangeError`, process crashed. Reported as
[jslicense/spdx-expression-parse.js#38](https://github.com/jslicense/spdx-expression-parse.js/issues/38).

---

## How it was found

The audit checklist from
[the previous post](https://zoahdev.github.io/blog/2026/08/17/how-to-audit-a-small-plugin.html)
said: pick a target that handles untrusted input. `spdx-expression-parse`
parses license strings from `package.json` — untrusted input by definition —
and its core is ~3 KB. The first three parser targets (smol-toml,
dsh-session-export, dsh-companion) were clean; this one was not.

The parser is recursive-descent:

```text
parseExpression → parseAtom → parseParenthesizedExpression → parseExpression → …
```

No depth limit anywhere in the cycle.

## The reproduction

```js
const parse = require('spdx-expression-parse')
parse('('.repeat(5000) + 'MIT')
// RangeError: Maximum call stack size exceeded
```

Measured locally:

| Input | Result |
|---|---|
| 1,000 `(` | normal parse error (`Expected ')'`) |
| 5,000+ `(` | `RangeError: Maximum call stack size exceeded` |

That is ~5 KB of input to crash the process. DoS only — no memory corruption,
no code execution — but for a library that parses untrusted license strings in
supply-chain tooling, a 5 KB crash is still a real bug.

## Scope

- v5.0.0 — verified locally.
- v3.0.1 — same recursive structure confirmed in source (this is the widely
  deployed line; npm itself pulls it in).

## Disclosure path

The repository has no `SECURITY.md` and no private vulnerability reporting
enabled, so the report went to a public issue with the PoC, affected versions,
impact, and a suggested fix (a nesting-depth limit, or an iterative parser).
If the maintainer publishes a security advisory, the report is credited.

## Why this matters

Four clean audits could have been "the audits found nothing" — instead the
methodology was the point: pick the right surface, verify the defense (here,
the *absence* of a depth limit), and reproduce before claiming. The finding is
small, honest, and reproducible. That is what security work looks like at the
small-package scale: not a headline RCE, but a steady stream of
verifiable bugs that supply-chain tooling actually hits.

---

## 中文摘要

第五个审计靶标（spdx-expression-parse，npm 生态广泛使用、npm 本身依赖的
SPDX 许可证解析器）出了第一个真实漏洞：递归下降解析器没有嵌套深度限制，
`parse('('.repeat(5000)+'MIT')` 就栈溢出（RangeError），约 5KB 输入即可打崩
进程——纯 DoS。v5.0.0 本机实测，v3.0.1 源码确认同一结构。仓库没有安全策略和
私有上报通道，所以走公开 issue（#38），附 PoC/影响版本/修复建议（加深度上限
或改迭代解析）。方法论兑现了：选对表面、验证防御（这里是"没有深度限制"）、
先复现再声称。小包安全研究就是这样——不是头条 RCE，而是可复现的真 bug。
