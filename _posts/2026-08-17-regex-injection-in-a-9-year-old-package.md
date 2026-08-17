---
layout: post
title: "Regex injection in a 9-year-old package: what stale code teaches us"
date: 2026-08-17
---

# Regex injection in a 9-year-old package

**Summary.** The second real finding of the audit series: a config parser for
the FIS build tool, last published in 2017 and untouched since, interpolates
config keys into a `RegExp` without escaping. Keys with regex metacharacters
either corrupt the generated output or throw a `SyntaxError` that kills the
build. Reported as
[huhuaaa/fis-parser-config#1](https://github.com/huhuaaa/fis-parser-config/issues/1).

---

## The target

After the first finding (spdx-expression-parse), the audit strategy shifted:
well-maintained packages are defensively coded, so the next targets were
**low-maintenance parsers nobody has looked at recently**. `fis-parser-config`
fit perfectly: a config-variable substitution plugin for Baidu's FIS build
tool, last published **nine years ago**.

## The bug

```js
var reg = new RegExp('__conf\\(' + i + '\\)', 'g')
```

`i` is a config key, interpolated into the pattern with no escaping. Regex
metacharacters in a key change what the pattern means:

| Config key | What the regex does | Consequence |
|---|---|---|
| `a+b` | `+` becomes a quantifier | `__conf(aab)` replaced, `__conf(a+b)` not — **output corrupted** |
| `a(` | unbalanced group | `SyntaxError` — **build crashes** |
| `a.b` | `.` matches any character | `axb`, `ayb` all replaced — **wrong matches** |

All three verified with a minimal reproduction before reporting:

```js
parse('__conf(a+b) __conf(aab)', 'x', { keys: { 'a+b': 'VALUE' } })
// → '__conf(a+b) "VALUE"'  ← the wrong token was replaced

parse('__conf(a( )', 'x', { keys: { 'a(': 'VALUE' } })
// → SyntaxError: Invalid regular expression: /__conf\(a(\)/g
```

## Why this is a vulnerability, not a nit

The trigger surface is config keys — which sounds trusted. But "trusted" breaks
in practice: shared config files, CI-injected values, configs generated from
untrusted inputs, tampered files in a compromised workspace. Once a key is
attacker-influenced, the result is either silent data corruption in generated
output or a hard crash — both in a build pipeline where failures cascade.

## The lesson: packages don't die, they just stop being maintained

The interesting thing about this finding is that it was *waiting*: the code
has been wrong since 2017, shipped in a package that still installs and runs
in build pipelines. Maintainers move on; code does not. The low-maintenance
corner of npm is not empty of bugs — it is full of bugs nobody has reported
because nobody has looked.

The fix is trivial and the report is polite: escape the key
(`key.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')`) or use plain string search,
since the match target is a fixed literal.

---

## 中文摘要

审计系列第二个真实发现：FIS 构建工具的配置解析插件（2017 年后未再发布，
9 年未维护）把配置键未转义拼进 `new RegExp(...)`——键含正则元字符时要么错误
替换输出（数据损坏），要么 `SyntaxError` 崩溃（DoS）。三种键型全部实测复现：
`a+b` 误替换 `__conf(aab)`、`a(` 直接崩、`a.b` 任意字符匹配。触发面是配置键，
看似可信，但共享配置/CI 注入/被篡改文件都会打破信任。核心教训：**包不会死，
只是没人维护**——低维护角落不是没有 bug，而是没人去看。修复很简单：
转义键名或改用字符串查找。
