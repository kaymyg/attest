# Threat Model and Scope

This document exists to prevent the project from drifting into an unwinnable
problem. It states what is being defended, against whom, what is explicitly out
of scope, and — most importantly — **one framing that was considered and
rejected**, so nobody re-proposes it later.

---

## The rejected framing: detection by marker

The intuitive version of this project is: *"identify an escaped or malicious AI
by markers in its traffic."* Fingerprint the request patterns, the timing, the
header entropy, the phrasing — and flag what looks synthetic.

**This is rejected as a primary control, and the reason comes from the
[AMES](https://github.com/kaymyg/ames) framework this project builds on.**

AMES's own L5 (behavioural fingerprinting) documentation states the limit
explicitly: an adversary who has learned the baseline paces its behaviour at
exactly the historical mean and evades the check entirely. That is why L5 is
**advisory only** in AMES — behavioural markers cannot carry an enforcement
decision against an adversary assumed to be adaptive, patient, and able to read
the detector's source.

Moving that detector from the host to a web server makes the problem strictly
worse, for three reasons:

1. **The adversary controls its entire side of the connection.** Headers,
   timing, TLS fingerprint, request ordering — all attacker-chosen.
2. **Every published marker becomes a target.** A detection technique that
   works is a technique that gets optimised away as soon as it matters.
3. **The base rate is hostile.** Legitimate agent traffic is exploding, so a
   detector tuned to catch synthetic-looking traffic will mostly catch
   legitimate agents, and operators will switch it off. A control that gets
   disabled is worse than no control, because it stays on the architecture
   diagram.

This is the same conclusion the CAPTCHA ecosystem reached the hard way.
Behavioural signals remain useful **as advisory input for human review** — the
posture AMES already takes with L5/L6 — and are never an authorisation
decision here.

---

## The accepted framing: proof of containment

Stop asking *"what are you?"* Ask *"can you prove you are authorised, right
now?"*

That converts an adversarial arms race into a cryptographic check whose
difficulty does not depend on how clever the adversary is.

The industry is already partway here. [Web Bot
Auth](https://datatracker.ietf.org/doc/html/draft-meunier-web-bot-auth-architecture)
(IETF, chartered 2026; backed by Cloudflare, Amazon, Akamai, OpenAI) gives
agents cryptographic identity via HTTP Message Signatures (RFC 9421): an
Ed25519 key, a `Signature-Agent` header, a published JWKS directory.

**But identity is not containment.** Web Bot Auth answers *"this request comes
from operator X's agent."* It does not answer:

- Is that agent still running inside its intended constraints?
- Did it escape its sandbox ten minutes ago and keep its signing key?
- Has its supervisor observed anything that should have stopped it?

An escaped agent is, by construction, an agent whose credentials are still
valid. **Signing keys survive escape.** That is the gap this project targets.

### The mechanism

The agent's runtime issues **short-lived signed attestations** of containment
state:

```
"agent  : <public key id>
 run    : <run id>
 gate1  : passing
 kill   : not tripped
 ledger : <checkpoint digest, height N>
 issued : <timestamp>
 expires: <timestamp + 30s>"          — signed by the supervisor
```

A website requires a fresh attestation alongside the Web Bot Auth signature. An
escaped agent fails the check **not because anyone detected the escape, but
because the supervisor stopped vouching for it.**

The security property is *absence of proof*, not *presence of evidence*. This
inverts the arms race: the adversary cannot forge a fresh attestation without
the supervisor's key, and cannot obtain one without actually still being
contained.

AMES already computes every input this needs — the ledger, the Gate 1 result,
the kill verdict, the checkpoint. The attestation is a signed summary of state
that exists today.

---

## Assets and adversaries

**Assets being protected:** the availability, integrity, and resource budget of
a public web service; its users' data; and any action the service performs on a
caller's behalf.

**Adversaries, in order of how well this defends against them:**

| Adversary | Defended? |
|---|---|
| Escaped agent presenting valid but stale credentials | **Yes** — cannot obtain a fresh attestation |
| Unattested agent (no runtime, no supervisor) | **Yes** — cannot produce an attestation at all |
| Attacker impersonating a known agent operator | **Yes** — by Web Bot Auth, which this composes with |
| Compromised supervisor signing false attestations | **No** — trust root failure, see below |
| Authorised agent behaving maliciously within its grants | **No** — see "identity is not intent" |
| Human attacker with no AI involvement | **Partially** — same as any bot-auth scheme |

---

## Explicitly out of scope

**Identity is not intent.** An authorised, genuinely contained agent can still
make requests that harm the service — scraping past its rate budget, exercising
an expensive endpoint, or acting on a poisoned instruction. No identity or
attestation system solves this. It falls back to least privilege, rate
limiting, and server-side anomaly detection. **This project must not claim to
distinguish a malicious call from a genuine one when both are properly
authorised.**

**Also out of scope:**

- Detecting AI-generated *content* (a different problem with different, weaker
  guarantees)
- Responding to an agent already established inside a network — that is
  incident response
- Alignment or intent modelling of any kind
- Any claim of absolute prevention

---

## Known hard problems

These are unsolved and should be treated as research questions, not
implementation details.

**The trust root.** Who signs attestations, and why should a website trust that
signer? A self-signed supervisor attestation proves nothing to a stranger. This
needs either a federated registry (the direction Web Bot Auth's Signature Agent
registry points) or hardware-rooted attestation (TPM/SEV-SNP/TDX), and the
choice is as much political as technical. **This is the hardest problem in the
project.**

**Adoption is chicken-and-egg.** The scheme is worthless until both agents and
sites implement it. The only realistic path is riding Web Bot Auth as an
*extension* — an additional header on an existing standard — rather than
competing with it.

**Privacy.** An attestation leaks operational detail about the agent's
operator: run identifiers, timing, possibly workload shape. [PACT's anonymous
token
approach](https://securityboulevard.com/2026/06/new-pact-protocol-could-help-sites-distinguish-user-backed-ai-agents-from-malicious-bots/)
is attacking a related version of this and is worth studying before designing
the attestation payload.

**Revocation latency.** A 30-second attestation window means up to 30 seconds
of access after containment fails. Shorter windows cost round-trips and
availability. There is no free choice here, only a stated tradeoff.

**Clock dependence.** Freshness checks require the verifier and issuer to agree
roughly on time. Clock skew becomes a security parameter.

---

## Success criteria

This project succeeds if a website can, in a single request, distinguish:

1. an agent under live, verifiable containment,
2. an agent with valid identity but no containment proof,
3. an agent with neither,

**without** inspecting behaviour, and **without** any assumption about how
sophisticated the caller is.

It fails if it becomes another heuristic detector, or if it claims to
distinguish good intent from bad.
