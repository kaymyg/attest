# attest — Proof of Containment for AI Agents

**A website cannot tell whether the agent calling it is still inside its
sandbox. This project makes that provable — so an escaped agent fails the check
automatically, not because anyone detected it, but because its supervisor
stopped vouching for it.**

Companion to [AMES](https://github.com/kaymyg/ames), which contains agents from
the inside. This is the other half: letting the outside world verify that
containment is still holding.

**Status: design stage. No implementation yet.** The threat model and scope are
written; the code is not. See [Status](#status).

---

## Introduction

An AI agent making an HTTP request looks like any other HTTP client. If that
agent has escaped its execution environment, it still holds whatever
credentials it was issued — **signing keys survive escape.** The website on the
receiving end has no way to know the difference.

That gap is not hypothetical. Roughly **80% of AI agents don't properly
identify themselves** to the sites they visit, and most sites cannot tell a
legitimate agent from an attacker imitating one. The IETF chartered a working
group on this in 2026.

## Purpose

Let a website verify, per request, that the agent calling it is **under live,
supervised containment right now** — without inspecting its behaviour, and
without any assumption about how sophisticated it is.

## The idea in one paragraph

The agent's runtime issues short-lived signed attestations of its own
containment state: *Gate 1 passing, kill switch untripped, ledger at checkpoint
N, expires in 30 seconds.* A website requires a fresh one alongside the agent's
identity signature. An escaped agent cannot obtain a new attestation, because
obtaining one requires still being contained. The security property is
**absence of proof**, not presence of evidence — which is what makes it
resistant to an adversary that can shape everything about its own traffic.

## What this is not

**It is not a detector.** No fingerprinting, no traffic-pattern heuristics, no
"does this look synthetic." That approach is rejected outright in
[`docs/threat_model.md`](docs/threat_model.md), for the reason AMES's own L5
documentation already establishes: an adversary that can read the detector
paces itself under the threshold. Behavioural signals are advisory input for
human review, never an authorisation decision.

**It does not distinguish good intent from bad.** An authorised, genuinely
contained agent can still behave badly within its grants. No identity or
attestation system solves that — it falls back to least privilege and rate
limiting. Claiming otherwise would be the project's most likely failure mode,
so it is ruled out in scope on day one.

## How it composes with existing work

This is an **extension to [Web Bot
Auth](https://datatracker.ietf.org/doc/html/draft-meunier-web-bot-auth-architecture)**,
not a competitor to it. That standard — IETF-chartered in 2026, backed by
Cloudflare, Amazon, Akamai and OpenAI, with support shipped in AWS WAF, Vercel,
Shopify and Akamai — already solves agent *identity* using HTTP Message
Signatures (RFC 9421).

| Question | Answered by |
|---|---|
| Which operator does this agent belong to? | Web Bot Auth (RFC 9421 signatures) |
| Is a human in the loop? | [PACT](https://securityboulevard.com/2026/06/new-pact-protocol-could-help-sites-distinguish-user-backed-ai-agents-from-malicious-bots/) (anonymous tokens) |
| **Is this agent still contained?** | **this project** |
| Is this agent behaving well? | *nobody — see scope* |

Riding an adopted standard is also the only realistic answer to the
chicken-and-egg problem: an additional header on a request that already carries
a signature costs far less to adopt than a new protocol.

## Relationship to AMES

AMES already computes every input an attestation needs — the tamper-evident
ledger, the Gate 1 structural result, the kill verdict, the checkpoint digest.
The attestation is a signed summary of state that exists today.

This repo therefore **depends on `ames`** rather than duplicating it. It is not
a fork; the codebases share almost nothing. AMES points inward at the contained
agent, `attest` points outward at everyone who has to trust it.

## Status

Design stage. `docs/threat_model.md` is written and is the authoritative scope
document — including the framing that was **considered and rejected**, recorded
so it doesn't get re-proposed.

Nothing is implemented. The hard problems are known and unsolved:

- **The trust root** — who signs, and why would a stranger trust them? Hardest
  problem in the project, and as much political as technical.
- **Adoption** — worthless until both sides implement it.
- **Privacy** — attestations leak operational detail about the operator.
- **Revocation latency** — a 30-second window means up to 30 seconds of access
  after containment fails. A stated tradeoff, not a free choice.

## Success criteria

A website can distinguish, in one request, an agent under live verifiable
containment / an agent with identity but no containment proof / an agent with
neither — without inspecting behaviour.

It fails if it becomes another heuristic detector, or claims to judge intent.

MIT.
