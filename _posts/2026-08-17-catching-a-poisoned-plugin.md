---
layout: post
title: "Catching a poisoned plugin before it runs: three layers and an honest threat model"
date: 2026-08-17
---

# Catching a poisoned plugin before it runs

**Summary.** Plugin ecosystems inherit npm's supply-chain problem: one
obfuscated `postinstall` can exfiltrate a user's credentials before anyone
reads the source. [dsh-poison-guard](https://github.com/zoahdev/dsh-poison-guard)
scans a DeepSeek Harness plugin before `dsh plugin add` with three layers —
AST analysis, a deobfuscation decoder, and regex fallback — while stating its
threat model honestly: **this is defense-in-depth, not a security boundary**.

---

## Three layers

**1. AST analysis** via NodeSecure JS-X-Ray (the SAST behind NodeSecure CLI):
variable tracing, dynamic-import resolution, obfuscator detection,
`eval` / `Function` / `vm` sinks, `data-exfiltration`,
`serialize-environment`, unsafe shell commands, and more.

**2. Deobfuscation decoder**: unpacks `atob()`,
`Buffer.from(..., "base64"/"hex")`, `String.fromCharCode(...)`, and
`\xNN` / `\uNNNN` escapes, then **re-scans the decoded strings** for hidden
credentials, URLs, and shell commands. This is the layer that catches the
"it's just base64, nobody will read it" attacks.

**3. Regex fallback**: for obvious literals, non-code files, and install-time
scripts (`prepare` / `postinstall` / `install` / `preinstall`) — the classic
`curl ... | sh` surface.

Rules carry their layer as a prefix: `ast/*` (JS-X-Ray),
`deobfuscated-*` (decoder), `install-script*` (manifest), unprefixed (regex).

## What it makes visible

| Severity | Examples |
|---|---|
| HIGH | data exfiltration, obfuscated require, eval/Function/vm, unsafe commands, decoded secrets/keys/commands, exfil combos |
| MEDIUM | env serialization, shady links, SQL injection, monkey-patching, prototype pollution, decoded URLs, network egress, child_process, install scripts |
| LOW | encoded literals, short identifiers, unsafe regex, weak crypto, env reads |

## The honest threat model

No static tool can catch **all** poisoning: detecting arbitrary malicious
behavior in arbitrary code is undecidable (Rice's theorem), and a determined
attacker can always craft an obfuscation a scanner cannot see through. What
the tool does is make the **cheap, high-volume attacks** visible to someone
who would never find them by reading source — hidden exfiltration URLs,
obfuscated `require("child_process")`, `eval` of base64 blobs, `process.env`
harvests, `.ssh` reads, install-time `curl | sh`.

The real boundary is the harness sandbox: keep untrusted plugins in
`workspace-write`, never `danger-full-access`. The last layer is provenance:
prefer verified, maintained, clearly-authored plugins. A scanner that claims
to be a security boundary would be exactly the kind of overclaim that destroys
trust — so this one explicitly says what it is not.

## Usage

```sh
# human-readable verdict
dsh-poison-guard scan ./some-plugin

# machine-readable, for CI gates
dsh-poison-guard scan ./some-plugin --json

# in-harness: the agent gains a plugin_scan tool
dsh plugin --profile web add dsh-poison-guard
```

Live demo: https://zoahdev.github.io/dsh-poison-guard/

---

## 中文摘要

插件生态继承了 npm 的供应链问题：一个混淆的 postinstall 就能在没人读源码
之前偷走凭据。dsh-poison-guard 在安装前用三层扫描：AST（JS-X-Ray：变量追踪、
动态 import、eval/Function/vm 汇、数据外发）、反混淆解码器（解 atob/base64/
fromCharCode/转义后**重新扫描解码串**找隐藏凭据和命令）、正则兜底（安装脚本
curl|sh）。威胁模型说得很直白：检测任意恶意行为不可判定（Rice 定理），
这是纵深防御不是安全边界；真正的边界是 harness 沙箱（workspace-write vs
danger-full-access）+ 来源可信度。检测规则按层前缀 ast/*、deobfuscated-*、
install-script* 分类，支持 --json 进 CI。
