# HITL Gate Kit (HGK): Product Requirements

**Source:** [HITL Gate Kit plan](https://github.com/WilsonWordsofWisdom/systemsideHITL/blob/research/hitl-gate-kit/docs/research/hitl-gate-kit-plan.md) in the prototype repository ([systemsideHITL](https://github.com/WilsonWordsofWisdom/systemsideHITL)). Where this PRD and the plan differ, this PRD wins and the difference is logged in [decisions.md](decisions.md).

## 1. Problem

Online services are now used by third-party AI agents: browser agents, MCP clients and personal agents. They can be manipulated (prompt injection, for example), can go further than the person wanted, or can hide that they are agents. Detecting them is unreliable: in the prototype's live testing, behaviour detection scored 0 against a modern browser agent.

What protects the person is a rule the server enforces: **an irreversible step (submit, pay, delete, change recovery details) can't complete without the account holder's own strong approval of exactly that step.** No reusable, technology-neutral way exists today for an organisation to build that gate and prove it holds.

## 2. Target users and jobs to be done

The kit is generic and sector-neutral. It is for any organisation that runs an online service: enterprise, company, startup or public sector.

| User | Job to be done |
|---|---|
| **System owner or product lead** | Decide which actions need human approval; understand the risk it reduces and what it costs |
| **Developer or integrator** (any stack: Java, .NET, Python, Node…) | Know exactly what to build, with integration patterns and examples for their stack |
| **Security assessor or auditor** | See the controls, threat model, conformance evidence and audit records |
| **Tester** | Prove the gate holds, including against live agents |

## 3. Core principles

1. **The gate and the approver are the control.** They are normative and tested.
2. **Agent detection is advisory.** It can make a decision stricter, never looser. The kit never ships detection weights as if they were proven.
3. **Specification first.** Adopters run many stacks, so a language-neutral specification and a black-box conformance suite reach more of them than an SDK alone.
4. **Proof over claims.** A conformance pass is only credible if the suite is shown to catch faults in a deliberately broken service.

## 4. Critical user journeys

Each journey gets a Gherkin feature file in `docs/tests/` (to be written; see [implementation-plan.md](implementation-plan.md), K1).

| ID | Journey | User | Done when |
|---|---|---|---|
| KJ1 | **Assess fit:** read the overview and threat model, choose the actions to gate | System owner | A list of gated actions, with a reason for each, drawn up using the guide's worksheet |
| KJ2 | **Integrate** in your own stack, following the specification and guide | Developer | Every gated action goes through one enforcement point that issues tickets and accepts approvals |
| KJ3 | **Prove conformance:** run the suite, get a report for the assessor | Tester, assessor | An HTML and JSON report showing pass, fail or skip for each requirement ID, at a stated HGK level |
| KJ4 | **Roll out:** observe, tune, then enforce, with observe mode time-boxed | System owner, developer | Enforcement is on; observe mode was announced while it ran and ended within 24 hours |
| KJ5 | **Test with live agents** using the playbook and results template | Tester | A results file for each agent and scenario, with ground truth and honest limitations |
| KJ6 | **Operate and audit:** read approval records, handle incidents | Operator, auditor | Every gated decision, approval and redemption can be traced in audit records that hold no secrets |

## 5. Success metrics and north star

- **North star:** the number of systems that pass **HGK Level 1** conformance in their test environment.

| Supporting metric | Target | Source |
|---|---|---|
| Time from first read to a passing Level 1 run (KJ2, KJ3) | Measured; tracked per adopter that reports it | Adopter feedback, issue template |
| Share of `broken-service` faults the suite catches | **100%** | CI conformance self-test |
| In live-agent tests on conforming systems, share of gated actions that ran without the account holder's approval | **0** | Live-agent results files |
| Defects the suite finds in real systems | Counted | Adopter reports, issues |
| Outside reviewers' rating of the guide | Measured, 1-5 | K5 read-through |

Analytics follow the HEART and north-star approach once v0.1 is released.

## 6. Scope

### 6.1 Releases

| Release | Contents |
|---|---|
| **v0.1** | 1. **Specification** (HGK-Core, -Approver, -Policy, -Actor, -Audit, -Rollout) with JSON Schemas and an OpenAPI description. 2. **Conformance suite** as a Docker image, with a `broken-service` example that proves it catches faults. 3. **Implementation guide** |
| **v0.2** | **TypeScript reference SDK** (`@hgk/*`) extracted from the prototype; OIDC step-up approver profile validated with two identity providers; `examples/express-minimal` |
| **v0.3** | CIBA with RAR approver profile; MCP profile tests (Level 3); Web Bot Auth strict and Content-Digest; the gateway pattern for older systems |

### 6.2 In scope

- The gate and tickets.
- Approver requirements and profiles (WebAuthn complete in v0.1; OIDC step-up and CIBA marked draft).
- The policy format, with a mapping to OpenID AuthZEN.
- Agent-facing responses for web, API and MCP; optionally Poppy.
- Audit records.
- Observe mode.
- The conformance suite.
- The implementation guide and live-agent test playbook.
- The reference SDK (v0.2).

### 6.3 Conformance levels

| Level | Requires | Typical use |
|---|---|---|
| **Level 1** | All of HGK-Core and POL-1, plus one approver profile meeting APR-1 to APR-5 and APR-9 to APR-11 | Minimum for production |
| **Level 2** | Level 1, plus HGK-Audit, CORE-15 (rate limits), CORE-16 (observe controls) and APR-6 | Recommended |
| **Level 3** | Level 2, plus the MCP profile and, where relevant, protocol identity checks (Web Bot Auth strict, Poppy) | Systems open to agent protocols |

The requirement IDs (CORE-*, POL-*, APR-*) are listed in [architecture.md](architecture.md) section 4 and become normative in the specification (K2).

## 7. Non-goals

- **An identity provider or account system.** The kit uses the organisation's own.
- **A bot-detection product.** Detection is advisory input, never the control that holds.
- **A web application firewall or edge product.** The gate sits at the action boundary inside the system.
- **Classifier weights presented as proven.** Each organisation tunes its own, or uses none.
- **Payment-specific standards** (Secure Payment Confirmation, AP2, Visa TAP). These stay optional notes.
- **A hosted service for adopters.**

## 8. Constraints

- **One developer.** v0.1 is about 4-6 weeks after the prerequisites (K0); v0.2 and v0.3 about 3-4 weeks each.
- **Developed in public** under the MIT licence (owner decision, 2026-11-27). Everything pushed is published at once, so the repository never receives secrets, live test data, results files or unreviewed research notes.
- **The licence for specification and guide text** must be chosen before the first content is pushed (K0).
- **Releases are human-gated,** never automatic on merge.
- **Support is best effort** in v0.1, with no service level.

## 9. Technical requirements

- **Specification:** Markdown with RFC 2119 keywords; JSON Schema (2020-12) for the gated response, outcomes, policy and audit record; OpenAPI 3.1 for the endpoints agents and approvers call.
- **Canonical form for payload hashes:** RFC 8785 (JCS) recommended, and required wherever two systems must produce the same hash.
- **Step-up challenge:** RFC 9470 `WWW-Authenticate` header on the gated response.
- **Conformance suite:** TypeScript with Playwright, shipped as a Docker image (`ghcr.io/<owner>/hgk-conformance`) so adopters need no Node toolchain. Configured with `hgk.config.yaml`. Outputs JUnit XML, JSON and HTML.
- **Reference SDK (v0.2):** TypeScript on Node. `@hgk/core` depends only on the platform's crypto and a schema validator; no framework or database code.
- **Hosting:** none. The kit runs in the adopter's environment and CI. No cloud resources are provisioned.
- **CI:** GitHub Actions with third-party actions pinned to commit SHAs; secret scan, dependency review, static analysis.
- **Release supply chain:** signed tags, npm provenance (Sigstore), CycloneDX SBOM, signed container image, aiming for SLSA build level 2.
- **Prototype dependency:** [systemsideHITL](https://github.com/WilsonWordsofWisdom/systemsideHITL) is the research testbed, the first conformance target and, from v0.2, the first SDK user.

## 10. Assumptions and risks

| Assumption or risk | Mitigation |
|---|---|
| The suite passes faulty implementations | Every `broken-service` fault switch must make its matching test fail |
| OIDC `nonce` binding or CIBA assumptions fail with real identity providers | Those profiles stay draft until validated with two providers; WebAuthn is the complete v0.1 profile |
| Adopters treat detection as the control | HGK-Actor is informative; the guide's threat model says so plainly; Level 1 must be safe with every agent `unknown` |
| Approver gaps H2-H4 (same-browser approval, approval fatigue) remain | APR-6 and APR-7 are "should" in Level 2; the guide gives the options |
| Developing in public exposes drafts and mistakes | Push only reviewed content; mark drafts; releases are reviewed and signed |
| Live-agent evidence is thin (one agent tested live) | More Phase 6 runs before the guide (K0 step 2); the guide states limits honestly |
| Supply-chain risk in a security kit | Pinned actions, provenance, SBOM, scans, human-gated releases |
| Scope creep | Specification first in v0.1; SDK in v0.2; profiles in v0.3 |
| Maintenance burden on one person | Small v0.1; best-effort support statement; conformance tests make outside contributions safer |

## 11. Smallest valuable slice

**HGK-Core plus the WebAuthn approver profile, with the conformance tests for them, run against both the prototype and `broken-service`.** That is enough for an organisation to build a Level 1 gate in any stack and prove it holds. The guide, Level 2 and Level 3 tests, and the SDK build on it.
