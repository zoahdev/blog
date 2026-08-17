---
layout: post
title: "Agents that remember: verification-driven self-evolution"
date: 2026-08-17
---

# Agents that remember: verification-driven self-evolution

**Summary.** Agents keep repeating expensive lessons: a crash fixed last week
gets re-debugged today, a Windows quirk gets rediscovered, an install trap
gets re-hit. [dsh-rule-evolve](https://github.com/zoahdev/dsh-rule-evolve)
turns that loop into a traceable pipeline —
`experience → learn → rules (AGENTS.md) → verify (real checks)` — where every
rule carries its source and its last verification result, and **nothing is
ever learned without being checked**.

---

## The loop

```text
experience (markdown)  →  learn  →  rules (AGENTS.md)  →  verify (real checks)
```

Concretely:

```sh
# 1. learn from a retrospective / troubleshooting doc
node scripts/dsh-evolve.mjs learn --from docs/troubleshooting.md --out experience.jsonl

# 2. generate rules
node scripts/dsh-evolve.mjs rules --experience experience.jsonl --out AGENTS.md

# 3. verify against the real repo — every entry gets stamped verified: true/false
node scripts/dsh-evolve.mjs verify --experience experience.jsonl --dir ./my-plugin
```

`verify` runs the real check pipeline (dsh-plugin-doctor `check <dir>` by
default, override with `--cmd`). A rule that fails its check is not trusted —
it is flagged.

## The self-improvement round (v0.2.0)

```sh
# reflect on a completed task
node scripts/dsh-evolve.mjs reflect --task "make my plugin pass its own checks" --result retro.md --out experience.jsonl

# evolve: verify against the real repo, install into a dsh profile, log one round
node scripts/dsh-evolve.mjs evolve --experience experience.jsonl --dir ./my-plugin --profile web --log EVOLUTION.md
```

The rules land in `<DSH_HOME>/profiles/web/AGENTS.md` (previous file backed
up), so the next session actually starts with the lessons. Every round is
logged:

```markdown
## Round 1 — 2026-08-15T…
- New rules: 5
- Verified: yes ✅
- Sources: examples/doctor-experience.md
- Command: dsh-evolve evolve --experience … --dir … --profile web
```

## Why verification gates are the whole point

There are two ways to let an agent "evolve":

1. **Evolve which code runs** — powerful, and dangerous without boundaries.
2. **Evolve which rules are trusted** — with a verification gate between
   "learned" and "applied".

The second is the direction that stays auditable: every rule has a source and
a last-verification timestamp; nothing enters the profile silently. Evolution
is a reviewed diff, not a mutation.

## Dogfood

The doctor repo passes its own check with five verified rules installed into a
profile (see `examples/demo/EVOLUTION.md`). The loop is not hypothetical — it
is the same repository that ships the checks, running the checks on itself.

---

## 中文摘要

agent 最大的浪费是重复踩同一个坑。dsh-rule-evolve 把"经验 → 规则 → 验证"
变成可审计流水线：每条规则带来源和最近一次验证结果，没有被真实检查过的东西
不会被信任。v0.2.0 支持 reflect → evolve 完整闭环：规则通过真实检查后安装进
dsh profile，下一轮会话直接带着教训启动，每轮写进 EVOLUTION.md。关键是方向：
不是"让代码自我修改"，而是"让被验证的规则自我进化"——进化是一次可 review
的 diff，不是一次变异。
