---
layout: post
title: "The bug that wasn't: how testing kept me from reporting a false vulnerability"
date: 2026-08-17
---

# The bug that wasn't

**Summary.** During the audit of `fast-xml-parser` v5.7.3, I found what
looked like a real bug: an `entity.regx` typo (should be `entity.regex`) in
`EntitiesParser.replaceEntitiesValue`, which should have made DOCTYPE entity
replacement silently never work — and possibly corrupt values by replacing the
literal text "undefined". Before filing, I ran the reproduction. The bug was
not in the active code path: `&foo;` was replaced correctly, and no data was
corrupted. This post is about why that near-miss matters.

---

## The code-level hypothesis

In `fast-xml-parser` v5.7.3's `src/v6/EntitiesParser.js`:

```js
replaceEntitiesValue(val) {
  if (typeof val === "string" && val.length > 0) {
    for (let entityName in this.docTypeEntities) {
      const entity = this.docTypeEntities[entityName];
      val = val.replace(entity.regx, entity.val);   // ← typo: regx, not regex
    }
    ...
  }
}
```

`addDocTypeEntities` stores the pattern under `regex`, but the replacement
loop reads `regx` — `undefined`. Two plausible consequences:

1. DOCTYPE-defined entities (`<!ENTITY foo "BAR">`) are never replaced;
2. `String.prototype.replace(undefined, x)` coerces `undefined` to the string
   `"undefined"` and replaces the **first literal "undefined" in the value**
   with the entity's replacement — silent data corruption.

That second consequence would have made this more than a nit: a parser
corrupting values that contain the word "undefined" is a real bug.

## The reproduction

```js
const { XMLParser } = require('fast-xml-parser')
const p = new XMLParser()

// Hypothesis 1: entity never replaced
p.parse('<!DOCTYPE r [<!ENTITY foo "BAR">]><r>&foo;</r>')
// → {"r":"BAR"}   ← replaced correctly!

// Hypothesis 2: literal "undefined" corrupted
p.parse('<!DOCTYPE r [<!ENTITY foo "BAR">]><r>undefined</r>')
// → {"r":"undefined"}   ← untouched!
```

Both hypotheses failed. The active parser (`src/xmlparser/`) handles DOCTYPE
entities correctly; the `regx` typo lives only in a **non-active source copy**
under `src/v6/`. A bug report filed from the code-level hypothesis alone would
have been wrong — and worse, it would have burned exactly the maintainer trust
that the audit exists to build.

## The three gates before reporting

1. **Code-level hypothesis** — "this looks wrong because X".
2. **Minimal reproduction** — run the smallest input that should trigger X,
   and also run the input that should *not*.
3. **Scope check** — is this code path actually active? Which entry point,
   which version, which build?

The reproduction is the gate that separates a finding from an opinion. In this
case it caught the difference between "source copy" and "active code path" —
the same difference that earlier produced a *real* finding
(spdx-expression-parse, issue #38) where the recursion was verified before
reporting.

## Why this is worth writing down

Audits produce false positives. The professional move is not to pretend they
never happen — it is to make them cheap and visible:

- the reproduction came before the report;
- the record shows the hypothesis, the test, and the correction;
- the maintainer was never asked to triage a bug that did not exist.

That is the actual trust model of security work at this scale: not "never
wrong", but "wrong cheaply, in private, with the evidence attached".

---

## 中文摘要

审计 fast-xml-parser v5.7.3 时看到一个 `entity.regx` 拼写错误（应为
`regex`），推理出两个后果：DOCTYPE 实体永不替换 + 值里的字面量 "undefined"
被悄悄替换成实体值（数据损坏）。**提交前先跑了复现**：`&foo;` 被正确替换成
"BAR"，字面量 "undefined" 原样保留——拼写错误只存在于非活跃的 `src/v6/`
源码副本，活跃解析器（xmlparser/）没有这个 bug。差点误报。纪律：报漏洞前过
三关——代码假设 → 最小复现（含"应该不触发"的对照）→ 作用域检查（这个代码
路径真的激活吗）。审计必然有假阳性，专业不是"从不犯错"，而是"错得便宜、
错得私有、证据随附"。
