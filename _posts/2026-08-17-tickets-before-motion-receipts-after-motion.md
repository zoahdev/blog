---
layout: post
title: "Tickets before motion, receipts after motion: permission infrastructure for physical AI"
date: 2026-08-17
---

# Tickets before motion, receipts after motion

**Summary.** [KineGrant](https://github.com/zoahdev/kinegrant-protocol) is an
open, auditable permission layer for physical AI: robots and agents must hold
a short-lived, one-time capability **before** an actuator moves, and produce
a signed, tamper-evident receipt **after** it does. Default-deny, offline
verifiable, standards-complementary — not a token and not a blockchain.

---

## The problem: physical action is binary today

Robots and AI agents are starting to touch the real world — opening doors,
moving arms, recording video. Most authorization in that world is
all-or-nothing: either the machine can actuate or it cannot, and after the
fact there is no way to prove what was authorized and what actually happened.

Software had this problem solved decades ago (capabilities, audit logs,
revocation). Physical AI inherits the *need* without inheriting the
*infrastructure*.

## The one-liner

> **Tickets before motion, receipts after motion.**

Before a machine acts, it needs a capability bound to exactly:

- who (the agent identity),
- what (the target and action),
- why (the policy decision and purpose),
- how long (short-lived, one-time, expiring).

If it isn't allowed, **it doesn't act** — default-deny, and the actuator does
not move. After it acts, there is a signed, privacy-minimized receipt proving
what was authorized and what happened.

## What it deliberately is not

KineGrant is not a token, not a blockchain, not robot middleware, not a motion
planner, and not a functional-safety system. It **complements** W3C ODRL, W3C
Web of Things, IEEE 7012, ROS 2 / SROS2, OPC UA, and Matter — it sits at the
authorization and accountability layer those standards don't cover.

The honest disclaimer is in the README: *do not use this implementation as the
sole safety control for real machinery.*

## The engineering body

- **KGP-001**: an open experimental spec (draft 0.1, stable wire format 1.0)
  with a separate [threat model](https://github.com/zoahdev/kinegrant-protocol/blob/main/spec/THREAT-MODEL.md).
- **Post-quantum signatures**: ML-DSA-65 for capability and receipt signing,
  so the audit trail does not rot as crypto ages.
- **Hardware**: ESP32-C3 firmware — the gate can live close to the actuator.
- **Offline verifier**: a zero-dependency browser verifier handles signed
  bundles, delegation chains, forbidden combinations, receipts, MPT evidence,
  fleet operations, and hardware evidence locally — nothing leaves the
  browser.
- **Reference implementation** v2.65.4, Apache-2.0, with `kinegrant-protocol`
  on PyPI and `kinegrant-js` on npm.
- **Reproducible evidence**: a `REPRODUCING.md` path plus OpenSSF Best
  Practices and Scorecard badges.

Try it without installing: [kinegrant.com/playground.html](https://kinegrant.com/playground.html).

## Why this matters now

Physical AI is moving faster than its permission model. The gap between "the
robot can do anything" and "the robot can prove it was allowed to do exactly
this" is where accidents, abuse, and liability live. A narrow, auditable,
standards-complementary permission layer is the unglamorous piece that makes
the rest of the field possible — and it has to be open so independent
reviewers can check the threat model, the crypto, and the gates.

---

## 中文摘要

KineGrant 是为物理 AI（机器人/硬件）做的开放、可审计的授权层：动作前必须持有
短时一次性 capability（绑定谁/什么/为什么/多久），默认拒绝；动作后产生签名、
防篡改的 receipt。它不是 token、不是区块链、不是安全功能系统，而是补 W3C
ODRL / WoT / IEEE 7012 / ROS2 / OPC UA 的授权与问责空档。工程面包括 KGP-001
规范 + 威胁模型、后量子 ML-DSA-65、ESP32-C3 固件、零依赖离线浏览器验证器，
参考实现 v2.65.4（PyPI + npm），并明确声明"不能作为真实机械的唯一安全控制"。
一句话：动作前给票，动作后给票根。
