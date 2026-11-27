# Decisions log (append-only)

Dated entries for key business and technical decisions. Earlier decisions that shaped the kit are in the prototype's [decisions.md](https://github.com/WilsonWordsofWisdom/systemsideHITL/blob/main/docs/decisions.md).

## 2026-11-27: Specification first; v0.1 is specification, conformance suite and guide
**Decision:** v0.1 ships the HGK specification, a black-box conformance suite and the implementation guide. The TypeScript reference SDK follows in v0.2.
**Why:** Adopters' stacks are mixed or unknown (Java, .NET, Python, Node). A TypeScript SDK alone would reach few of them; black-box tests let any implementation prove it holds.
**Alternatives:** SDK first (faster for Node adopters, excludes the rest); specification only (no way to prove conformance).

## 2026-11-27: The gate and approver are normative; agent detection is advisory
**Decision:** HGK-Core and HGK-Approver are normative. HGK-Actor is mostly informative; detection can only make a decision stricter. No detection weights are shipped as proven.
**Why:** Behaviour detection scored 0 against a modern browser agent in the prototype's Phase 6 testing, while the server-side gate held.
**Alternatives:** Ship tuned weights (unproven, and invites treating detection as the control).

## 2026-11-27: Approvers are pluggable, behind required properties
**Decision:** APR-1 to APR-5 define what any approver must prove. WebAuthn is the complete v0.1 profile; OIDC step-up and CIBA are draft until validated with real identity providers.
**Why:** Organisations already have identity providers and should be able to use them; the properties, not the mechanism, are what make approval trustworthy.

## 2026-11-27: Developed in public, MIT licence for code
**Decision:** This repository is public. Nothing is pushed that contains secrets, live test data, results files or unreviewed research notes. Releases stay a reviewed, human-gated step.
**Why:** Owner decision; supersedes the earlier plan to share privately first.
**Open:** the licence for specification and guide text (MIT or CC BY 4.0) must be chosen before the first content is pushed (implementation plan O1).

## 2026-11-27: The conformance suite must prove it catches faults
**Decision:** `examples/broken-service` has one switch per requirement fault; CI fails unless the suite catches every one.
**Why:** A suite that passes faulty implementations gives assessors false confidence. A mutation-style self-test makes a "pass" credible.

## 2026-11-27: No hosted service; no cloud resources
**Decision:** The kit runs in the adopter's environment and CI. Nothing is provisioned.
**Why:** Hosting is a non-goal; the gate belongs inside each organisation's own system.

## 2026-11-27: Security review of the architecture; high-severity fixes adopted
**Decision:** Seven high-severity findings (SA-1 to SA-7) from a review of `architecture.md` are fixed in the specified design:
- **SA-1:** the gap between checking a ticket and running the action → new CORE-17 (check, consume and execute as one atomic unit).
- **SA-2:** the policy could let `human` be looser than `unknown` → new POL-1 start-up check.
- **SA-3:** the payload hash didn't cover enough → CORE-5 now covers the method, route, query and the parsed body, and refuses duplicate JSON keys.
- **SA-4:** enrolment and recovery could bypass the gate → APR-5 extended; new APR-9 (credential and recovery changes are `critical`, the account holder is notified, a cooling-off period).
- **SA-5:** approval method downgrade → new APR-11 (allowed profiles fixed in the ticket); the OIDC profile requires a phishing-resistant `acr`/`amr` and a fresh `auth_time`.
- **SA-6:** a relayed link or QR code can phish → APR-4 shows the origin; APR-6 requires an origin-bound credential on the approving device, or the cross-device BCP mitigations.
- **SA-7:** no approval page hardening → new APR-10.

POL-1, APR-9, APR-10 and APR-11 join Level 1.
**Why:** Each was a way around the gate that didn't need to break its cryptography. Sources: OWASP Transaction Authorization Cheat Sheet (sections 1.2-1.4, 2.2, 2.6, 2.8), PSD2 RTS Article 5 (dynamic linking), RFC 7493 and RFC 8785, RFC 8176, IETF draft-ietf-oauth-cross-device-security.
**Impact:** the prototype will likely need changes in K4 to pass Level 1/2 (atomic execute with consume, reserved credential actions in its policy, new-credential notification, approval page headers). Its current policy already meets POL-1.
**Deferred:** medium- and low-severity findings (observe-mode controls, the external policy engine floor, audit integrity, agent-message templates, sender-constrained tokens, approval flooding, guessing at the approval step, test-mode hooks, the 202 status, image hardening, a threat-model section) await an owner decision.

## 2026-11-27: Medium-severity security gaps noted (not yet adopted)
**Status:** open. These came from the same architecture review as SA-1 to SA-7. They are recorded so they aren't lost; each needs an owner decision before it changes the architecture.

| ID | Gap | Proposed fix | Affects |
|---|---|---|---|
| SA-8 | **Same-browser computer-use agents:** an agent driving the person's browser can operate the approval page and the passkey prompt, and type a PIN it has seen (prototype gaps H2, H3) | At Level 2, approving on a different device becomes a **must** whenever there is agent evidence; prefer biometric user verification; state the remaining risk in the guide | APR-6, Level 2 |
| SA-9 | **Observe mode turns enforcement off:** the switch itself is a bypass if it can be reached | Only set through configuration or deployment by an admin, never by a request; never applies to `critical` actions or credential management; turning it on is audited and alerted | CORE-16 |
| SA-10 | **An external policy engine (AuthZEN) can loosen decisions** | The final decision is the stricter of the local policy and the engine's answer; engine errors or timeouts fail closed; calls to it are authenticated | §2, HGK-Policy |
| SA-11 | **Audit records:** an unsalted hash of an IPv4 address can be reversed (only 2³² values); records aren't tamper-evident; a failed audit write isn't handled; `evidence` may hold personal data | Keyed HMAC with key rotation; a hash chain or append-only store; a failed audit write blocks the gated action at Level 2; data-protection and retention notes; minimise `evidence` | HGK-Audit, data model |
| SA-12 | **The agent-facing message can carry prompt injection:** user-controlled text (a resource name, for example) placed in `message_for_user` reaches the agent | Messages built from fixed templates with no user-controlled text; the approval summary is never returned to the agent | CORE-14 |
| SA-13 | **A stolen API or MCP bearer token can redeem a ticket** as the "same client plus user" | Level 3 recommends sender-constrained tokens (DPoP, RFC 9449, or mutual TLS); check token audience; never accept tokens not issued to this server (MCP "token passthrough" rule) | CORE-10, Level 3 |
| SA-14 | **Approval fatigue and flooding** (OWASP agentic threat T10, "Overwhelming Human in the Loop") | A new ticket for the same action and resource replaces older pending ones; the pending-approval cap becomes a **must** at Level 2; declines raise alerts (CISA push-bombing guidance); rate limits group IPv6 addresses by /64 | APR-7, CORE-15 |
| SA-15 | **Guessing at the approval step:** repeated failed approvals aren't bounded per ticket | After N failed approval attempts the ticket is rejected and the process restarts (OWASP Transaction Authorization 2.4) | APR-3, CORE-15 |

**Why it's recorded now:** none of these breaks the gate's core binding, but each weakens a control around it. Low-severity findings stay as listed under "Deferred" in the previous entry.
