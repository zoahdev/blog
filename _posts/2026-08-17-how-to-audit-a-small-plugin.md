---
layout: post
title: "How to systematically audit a small plugin (and why 'found nothing' still counts)"
date: 2026-08-17
---

# How to systematically audit a small plugin

**Summary.** Plugin ecosystems inherit supply-chain risk, but most small
plugins are not vulnerable — which means "found nothing" is a normal, honest
audit result, not a failure. This post is the exact checklist and workflow
used on three real targets (a TOML parser, a session exporter, and a
session-companion plugin). All three came back clean; the discipline is the
deliverable.

---

## The checklist

For a small package (< 2k lines of core logic) that handles untrusted input,
check these in order:

1. **Path traversal** — user-controlled strings reaching `fs` paths (`..`,
   separators, Windows reserved names, trailing dots, length caps).
2. **Prototype pollution** — recursive merges or key assignment where
   `__proto__` / `constructor` / `prototype` is not an own-property check.
3. **Command injection** — `child_process` with string-built commands or
   user input in shell contexts.
4. **SSRF** — fetching user-supplied URLs without scheme/host restrictions.
5. **ReDoS** — regexes with nested quantifiers on attacker-controlled input.
6. **Unsafe execution** — `eval`, `new Function`, dynamic imports of
   untrusted strings, unsafe deserialization.
7. **Secret handling** — hardcoded keys, API keys in logs/errors, weak crypto
   (ECB, fixed IV, missing auth tag).

## The workflow

1. **Pick the surface, not the README.** A parser, an exporter, a template
   engine, or a URL fetcher is a better target than a UI plugin.
2. **Read the risky files first.** For each target: crypto, HTTP, PDF/HTML
   escaping, path handling, child_process. Then grep the whole repo for
   `child_process`, `exec(`, `eval(`, `new Function` — zero hits is itself a
   data point.
3. **Verify defenses, don't just note their absence.** Seeing `__proto__` in
   a diff is not a finding: check whether it is handled with
   `Object.defineProperty` + `Object.hasOwn` (correct) or raw assignment
   (vulnerable). Seeing `escapePdfText` matters less than *what it escapes*:
   `(` `)` `\` and control chars as octal = correct; only `<` `>` = weak.
4. **Record files + lines.** An audit trail that names the file, the line,
   and the defense is worth more than a verdict.
5. **Report only real findings.** A low-severity observation (e.g., a
   symlink-following export directory) is a *note to the maintainer*, not a
   security advisory. Keep the bar high: GHSA-grade findings only.

## What three real audits found

| Target | Surface | Result |
|---|---|---|
| smol-toml | TOML parser | clean — `__proto__` guarded via `defineProperty`+`hasOwn`; anchored linear regexes; zero eval/child_process |
| dsh-session-export | session exporter | clean — `safeFilePart` strips unsafe chars/reserved names/trailing dots with a dedicated filename test; `uniquePath` never overwrites |
| dsh-companion | session companion | clean — AES-256-GCM correct (random IV, auth tag, length checks); router prefix-exact with 8MB/30s limits; PDF escaping correct; zero `child_process`/`exec`/`eval` in the whole repo |

All three are defensively implemented. That is not luck — it is what
decent small packages look like, and it is exactly why "found nothing" must
be an acceptable outcome. The alternative — shipping a finding that does not
hold up to review — destroys the trust the audit was supposed to build.

## Why "found nothing" still counts

1. It is **honest**: most tools are not vulnerable.
2. It builds a **corpus of verified-clean packages** — positive records that
   later audits (and users) can rely on.
3. The **discipline is the deliverable**: a reproducible checklist, named
   files and lines, and a defensible verdict. That survives even when the
   vulnerability count is zero.

---

## 中文摘要

小插件大多没有漏洞，"没发现"是正常且诚实的审计结果。这篇是三个真实靶标
（TOML 解析器 / 会话导出 / 会话伴侣插件）用的清单与流程：路径遍历、原型污染、
命令注入、SSRF、ReDoS、危险执行、密钥处理七项；先读风险文件（crypto/http/
pdf/路径），再全仓搜 child_process/exec/eval；验证防御而不是只看有没有出现
关键词（`__proto__` 要确认是 defineProperty+hasOwn 而不是裸赋值；转义要确认
转了什么）。三个靶标全部干净。关键原则：只报站得住的发现，低危观察写给
维护者而不是当 advisory；"没发现"也是资产——审计痕迹（文件+行号+防御说明）
本身就是交付物。
