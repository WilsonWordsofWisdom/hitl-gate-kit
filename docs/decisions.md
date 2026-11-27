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
