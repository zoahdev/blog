---
layout: post
title: "196 read-only tools, zero API keys: the discipline behind the catalog"
date: 2026-08-17
---

# 196 read-only tools, zero API keys

**Summary.** [dsh-github-intelligence](https://github.com/zoahdev/dsh-github-intelligence)
is a DeepSeek Harness plugin with 196+ read-only developer-intelligence tools
across 16 ecosystems (GitHub, GitLab, Gitee, npm, PyPI, crates.io, Docker Hub,
Hugging Face, Hacker News, Stack Overflow, Reddit, dev.to, RubyGems, NuGet,
the Go module proxy, ArXiv) — no API key required. This post is about the
rule that made it trustworthy: **a tool does not exist until its real API call
has been verified live**, and an API that looks public but is not gets dropped,
not papered over.

---

## The rule: verify live, then ship

Every tool in the catalog is backed by a real, anonymous HTTP call to the
ecosystem's public API. No mocks, no fixtures, no "should work" entries. A new
ecosystem only joins after:

1. a real anonymous request succeeds against the documented endpoint;
2. the response parses into a stable shape;
3. the tool returns that shape to the agent;
4. rate-limit behavior is understood (TTL caching added where the API is
   aggressive).

## The counter-example: Bitbucket

Bitbucket's Cloud API documents anonymous access. The verification run
discovered the opposite: **anonymous requests now return 404/410**. The API
is publicly documented and effectively closed. So Bitbucket was deliberately
**not** added — with a note explaining why. That decision is in the README to
this day:

> Bitbucket 已探测但故意不收录：公共 Cloud API 匿名访问现在返回 404/410——
> 这是"文档写着公开、实际已关"的坑，真实 API 验证拦住了它。

This is the whole point of the discipline: a catalog is only as honest as its
least-verified entry, and one fake tool poisons every downstream agent.

## Read-only as a security posture

All 196+ tools are read-only. The plugin never mutates a user's data through
the API surface — no commits, no issues, no releases. For an agent harness,
read-only intelligence tools are the difference between "a tool that helps"
and "a tool that needs its own permission model".

## Runtime discoverability

The catalog ships a `github_help` tool: call it and the agent learns what the
other 195 tools do and when to use them. 196 tools is too many to memorize;
it is not too many to *discover*. The plugin also features:

- unified tool style and one shared TTL cache;
- cancellation on every request;
- UI cards for the web surface;
- verified against `dsh` 0.1.0-rc.6, Node 24 / pnpm 11, CI green.

## Why this matters beyond one plugin

Agent tooling is entering the "API marketplace" era, and the bottleneck is no
longer writing a wrapper — it is **knowing which wrapper is real**. A catalog
that documents its own verification discipline (and its rejections) is a
small but real contribution to that trust problem.

---

## 中文摘要

dsh-github-intelligence 的 196+ 只读工具、16 大生态、零 API key，核心纪律是
"工具不存在直到真实调用被验证过"：每个工具背后都是真实匿名 HTTP 调用，
新生态必须过四步（真实请求成功 → 响应结构稳定 → agent 可消费 → 限流行为
清楚）。反面案例是 Bitbucket：文档写着公开、实测匿名访问 404/410，所以
故意不收并写明原因。全部工具只读、不碰用户数据；`github_help` 让 agent
运行时发现工具而不是硬记 196 个名字。在"API 市场"时代，能诚实记录自己
拒绝过什么的目录，才值得被信任。
