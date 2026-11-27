# HITL Gate Kit: Architecture

This document describes two things:
1. **The architecture the kit specifies:** how a conforming gate fits into an organisation's own system (sections 2-5).
2. **The architecture of the kit itself:** specification, conformance suite, guide and SDK, and how they relate (sections 6-9).

## 1. The four layers

Each layer can be used on its own.

| Layer | What it is | Who uses it | Release |
|---|---|---|---|
| **Specification (HGK)** | Requirements in RFC 2119 language, JSON Schemas, OpenAPI for the endpoints agents and approvers call | Developers, assessors | v0.1 |
| **Conformance suite** | Black-box HTTP and browser tests run as a Docker image against the adopter's test environment; JUnit, JSON and HTML report | Testers, assessors | v0.1 |
| **Implementation guide** | Threat model, choosing actions, integration patterns per stack, rollout, live-agent playbook, limitations | Everyone | v0.1 |
| **Reference SDK** (TypeScript) | `@hgk/*` packages extracted from the prototype | Node developers; a model for ports | v0.2 |

```mermaid
flowchart TB
  SPEC[Specification<br/>spec/ + schemas/]
  CONF[Conformance suite<br/>conformance/]
  GUIDE[Implementation guide<br/>guide/]
  SDK[Reference SDK<br/>packages/ - v0.2]
  BROKEN[broken-service<br/>examples/]
  PROTO[systemsideHITL prototype<br/>external repo]

  SPEC -- requirement IDs and schemas --> CONF
  SPEC -- normative text --> GUIDE
  SPEC -- implemented by --> SDK
  CONF -- must fail every fault in --> BROKEN
  CONF -- must pass Level 2 on --> PROTO
  CONF -- must pass Level 2 on --> SDK
  PROTO -- controls and evidence --> SPEC
  PROTO -- code extracted into --> SDK
```

## 2. How a gate fits into an organisation's system

```mermaid
flowchart LR
  subgraph Callers
    H[Person in a browser]
    BA[Browser agent]
    MA[MCP or API agent]
  end
  subgraph Your system
    R[Action endpoint<br/>submit, pay, delete...]
    G[HITL gate<br/>decision and enforcement point]
    P[(Policy<br/>actions x actor classes)]
    TS[(Ticket store<br/>single use, bound to payload)]
    AU[(Audit records)]
    A[Approval page<br/>on your own origin]
    S[Service logic]
  end
  subgraph Approver
    WA[WebAuthn passkey]
    IDP[Your identity provider<br/>OIDC step-up or CIBA]
  end
  H --> R
  BA --> R
  MA --> R
  R --> G
  G --> P
  G --> TS
  G --> AU
  G -- allow, or a valid approved ticket --> S
  G -- gate: approval needed, with ticket and link --> R
  H -- opens the link --> A
  A --> WA
  A --> IDP
  A -- approved --> TS
```

**Boundaries:**
- The gate sits **at the action boundary inside the system**, after authentication and before the service logic. It is not an edge product.
- The organisation keeps its own identity provider, sessions and database. The kit defines what the gate needs from them (section 7).
- An organisation with an existing policy decision point (OPA, Cedar, an AuthZEN service) can make the `allow / gate / deny` decision there. **Tickets and approval always stay in the gate; the policy engine never approves.**

## 3. A gated action, end to end

```mermaid
sequenceDiagram
  autonumber
  participant C as Requester (agent or person)
  participant G as Gate
  participant T as Ticket store
  participant P as Person (account holder)
  participant AP as Approval page + approver
  participant S as Service logic

  C->>G: POST /applications/42/submit
  G->>G: classify actor, look up policy, check ownership
  G->>T: create ticket bound to user, action, resource,<br/>payload hash and stored-state hash
  G-->>C: 403 requires_human_approval<br/>WWW-Authenticate (RFC 9470), ticket_id, continue_url
  C-->>P: relays message_for_user and link
  P->>AP: opens continue_url (configured origin only)
  AP->>T: read summary from the ticket record
  AP->>P: show summary and confirmation code
  P->>AP: approve with user verification<br/>(challenge bound to ticket and payload hash)
  AP->>T: pending → approved
  C->>G: retry with HITL-Ticket: <id>
  G->>T: same requester? same payload and state? not expired?<br/>atomic approved → consumed
  G->>S: run the action once
  S-->>C: 200
  Note over G,T: every step writes an audit record
```

## 4. Requirements the specification defines

These IDs are proposals; each traces back to a prototype control. They become normative text in K2.

### 4.1 HGK-Core (normative)

| ID | Requirement (summary) | Prototype control |
|---|---|---|
| CORE-1 | Every state-changing action passes one enforcement point; no route or tool bypasses it | C1, T11 |
| CORE-2 | Decision (`allow`, `gate`, `deny`) made on the server from a declared policy; unlisted, non-read-only actions are denied | C2, C4 |
| CORE-3 | Fail closed: an error gives `gate` (or `deny` for critical actions), never `allow` | C3 |
| CORE-4 | Resource ownership checked on every access | C5 |
| CORE-5 | Ticket bound to user, action, resource, request hash (minus named volatile fields) and stored-state hash, using a deterministic canonical form (RFC 8785 recommended). Input that can't be canonicalised is refused | T1 |
| CORE-6 | Ticket IDs carry at least 128 bits from a CSPRNG | T4 |
| CORE-7 | Lifetime at most 10 minutes, checked on read and on use | T3 |
| CORE-8 | States `pending → approved → consumed`; `pending → rejected / expired`; nothing else | T5 |
| CORE-9 | Used exactly once, atomically; concurrent attempts produce one winner | T2 |
| CORE-10 | Only the requester redeems (same web session, or same OAuth client plus user); others get `wrong_session` | T6 |
| CORE-11 | Changed payload or stored state after approval gives `payload_mismatch` | T1 |
| CORE-12 | The gated response (section 4.2) | FR21, FR24 |
| CORE-13 | Approval links and redirects built from configured origins only | T7, S7 |
| CORE-14 | Messages to agents are declarative and addressed to the person; no model-directed instructions, no secrets | P1-P3 |
| CORE-15 | Ticket creation and approval attempts rate limited per user and per network address | T9 |
| CORE-16 | Observe mode needs a second switch and an end time at most 24 h ahead; announced while on; enforcement resumes by itself | C6, H6 |

### 4.2 The gated response (CORE-12)

- **Status:** `403` by default; `401` or `202` allowed as a deployment choice.
- **Header:** `WWW-Authenticate: Bearer error="insufficient_user_authentication", acr_values="<deployment URN>"` (RFC 9470).
- **Body** (JSON Schema in `schemas/`): `status: "requires_human_approval"`, `action`, `ticket_id`, `continue_url`, `message_for_user`, `resume { poll, retry_with_header, expires_at }`.
- **Other outcomes:** `denied`, `rate_limited`, `ticket_invalid` with reason `not_found | wrong_session | not_approved | rejected | expired | consumed | payload_mismatch`.
- **Retry:** `HITL-Ticket: <id>` header; `X-HITL-Ticket` accepted for compatibility; forms may send `hitl_ticket`. Header name confirmed in K2.
- **Status endpoint:** `GET <poll>` returns the ticket state to its requester only.
- **MCP profile:** the same body as a tool result, or MCP URL elicitation. Gated tools say so in their descriptions.
- **Poppy profile (optional):** the ticket maps to an `operation`; the agent's own approval never completes a gated step.

### 4.3 HGK-Approver

| ID | Requirement | Gap addressed |
|---|---|---|
| APR-1 | Approval cryptographically bound to this ticket and this payload hash | W2 |
| APR-2 | Approver is the ticket's account holder, with user verification | W1, W3 |
| APR-3 | An approval assertion can be used once | W2 |
| APR-4 | Approval page shows a summary from the ticket record, on the organisation's origin | T8, W4 |
| APR-5 | A password alone can't enrol an approver credential; the first needs out-of-band proof, later ones need an existing credential | W12, H1 |
| APR-6 (should) | With agent evidence or suspected handover, approve on a different device or session | H2, H3 |
| APR-7 (should) | Confirmation code on both screens; limit pending approvals; cool down after a decline | H4 |
| APR-8 (may) | Attestation policy where the population allows it | H5 |

| Profile | Binding | Status |
|---|---|---|
| **WebAuthn** | Challenge `SHA-256(ticket_id ‖ payload_hash ‖ nonce)`, `userVerification: required`, origin and RP ID checked, credential belongs to the ticket's user | Complete (proven in the prototype) |
| **OIDC step-up** | `acr_values`, `max_age=0`, `nonce = base64url(SHA-256(ticket_id ‖ payload_hash ‖ server nonce))`; verify signature, `iss`, `aud`, `nonce`, `acr`, `auth_time` | Draft until validated with two identity providers |
| **CIBA with RAR** | `binding_message` with the confirmation code; `authorization_details` naming action and resource | Draft (v0.3) |

### 4.4 HGK-Policy, HGK-Actor, HGK-Audit, HGK-Rollout

- **Policy (normative):** YAML or JSON with a JSON Schema: `read_only` actions; per-action decisions for each actor class (`human`, `declared_agent`, `suspected_agent`, `unknown`); `critical` actions; gate settings. Loaded at start-up, read-only; invalid policy stops start-up. AuthZEN mapping: subject = user plus actor class; action; resource = type plus ID; context = route and evidence.
- **Actor (mostly informative):** normative parts are the classes, the evidence precedence (protocol identity > self-declaration > behaviour score > default), **escalate-only** within a session, and detection **never loosens** a decision. Signals are informative, with an explicit warning that they may score 0 for modern agents.
- **Audit (normative minimum):** a record for every decision, ticket change, approval, decline, redemption, denial and observe-mode decision. Never secrets, tokens, enrolment codes or assertions.
- **Rollout (informative):** observe (time-boxed) → tune → freeze → enforce. Detection rates only reported after the freeze.

## 5. Data model

The entities a conforming gate needs. The specification defines fields and invariants, not a database schema; `@hgk/postgres` (v0.2) gives one concrete schema.

```mermaid
erDiagram
  USER ||--o{ TICKET : "owns"
  USER ||--o{ APPROVER_CREDENTIAL : "enrols"
  TICKET ||--o| APPROVAL_CHALLENGE : "has one live"
  TICKET ||--o{ AUDIT_RECORD : "traced by"
  POLICY ||--o{ TICKET : "decides"

  TICKET {
    string id "≥128-bit random"
    string user_id
    string requester "session id, or OAuth client + user"
    string action
    string resource "type + id"
    string payload_hash "canonical form, volatile fields excluded"
    string state_hash "stored-state snapshot"
    string state "pending | approved | rejected | expired | consumed"
    time created_at
    time expires_at "≤ 10 min"
  }
  APPROVAL_CHALLENGE {
    string ticket_id
    string nonce "single use, server stored"
    string profile "webauthn | oidc | ciba"
  }
  APPROVER_CREDENTIAL {
    string user_id
    string profile
    string enrolled_via "out-of-band code | existing credential"
  }
  AUDIT_RECORD {
    time ts
    string route
    string action
    string user_id
    string session_or_client
    string actor_class
    string evidence
    string decision
    string ticket_id
    string outcome
    string reason
    string ip_hash
  }
  POLICY {
    list read_only
    map decisions "action x actor class"
    list critical
    map gate_settings
  }
```

**Ticket state machine (CORE-8):**

```mermaid
stateDiagram-v2
  [*] --> pending: gate decision
  pending --> approved: approver succeeds (APR-1..5)
  pending --> rejected: person declines
  pending --> expired: lifetime passes
  approved --> consumed: requester redeems, atomic (CORE-9)
  approved --> expired: lifetime passes
  rejected --> [*]
  expired --> [*]
  consumed --> [*]
```

**State ownership:** the gate alone writes tickets and challenges. The approval page moves a ticket from `pending` only through the approver. The policy is read-only at runtime. Audit records are append-only.

## 6. Failure handling

| Failure | Behaviour |
|---|---|
| Classifier or policy error | Fail closed: `gate`, or `deny` for critical actions (CORE-3) |
| Invalid policy at start-up | Start-up stops |
| Payload can't be canonicalised (`NaN`, lone surrogates, non-JSON values) | Request refused, never hashed (CORE-5) |
| Concurrent redemption | Atomic conditional update; one winner, others get `consumed` (CORE-9) |
| Requester loses its live status stream | Polling is the contract; streaming is a convenience |
| Approval storm | Rate limits (CORE-15) and pending-approval cap (APR-7) |
| Observe mode left on | Ends by itself at its end time, at most 24 h (CORE-16) |

## 7. Adopter ports (reference SDK, v0.2)

What an organisation supplies to `@hgk/core`. The specification describes the same needs in language-neutral terms for other stacks.

| Port | What it supplies |
|---|---|
| `ActionRegistry` | Per gated action: name, resource ID, a precondition check returning the stored-state snapshot and preview, volatile fields |
| `SessionAdapter` | User, session or client ID, sign-in method, an approval-only session for the approval page |
| `Storage` | Atomic conditional updates for tickets and challenges; rate-limit counters; audit records |
| `Approver` | One of the profiles in section 4.3 |
| `AuditSink` | Database, log pipeline or SIEM |
| `Clock` | Testable time |

```mermaid
flowchart LR
  core["@hgk/core<br/>policy, classifier framework, hash,<br/>ticket state machine, guard, responses"]
  webauthn["@hgk/webauthn"] --> core
  oidc["@hgk/oidc"] --> core
  pg["@hgk/postgres"] --> core
  express["@hgk/express"] --> core
  next["@hgk/next"] --> core
  signals["@hgk/signals<br/>advisory"] --> core
  mcp["@hgk/mcp"] --> core
```

`@hgk/core` depends only on platform crypto and a schema validator. No framework or database code.

## 8. Conformance suite

```mermaid
flowchart LR
  subgraph Docker image hgk-conformance
    RUN[Runner<br/>TypeScript + Playwright]
    DRV[Approver drivers<br/>webauthn-virtual, manual,<br/>oidc-mock stretch]
    MOCK[Mock IdP<br/>throwaway keys at start-up]
    REP[Reporter<br/>JUnit, JSON, HTML]
  end
  CFG[hgk.config.yaml<br/>base URL, two test accounts,<br/>gated actions, sign-in, driver]
  TGT[Target system<br/>TEST environment only]
  CFG --> RUN
  RUN --> DRV
  DRV --> MOCK
  RUN -- HTTP and browser --> TGT
  DRV -- approve tickets --> TGT
  RUN --> REP
```

- **Black-box:** the suite only sees HTTP and the browser. The target exposes test-mode hooks: an enrolment code for a test passkey, a short ticket lifetime, an audit export.
- **Safety:** refuses a base URL marked as production unless explicitly overridden.
- **Test catalogue:** about 30 tests in v0.1, grouped as gate coverage, tickets, responses, approver, limits and modes, audit (Level 2) and MCP (Level 3). Each test names the requirement IDs it covers.
- **Self-test:** `examples/broken-service` has one switch per fault (allow reuse, skip the payload check, accept any account's approval, build URLs from the request, and so on). CI runs the suite against each switch and fails if any fault goes undetected.

## 9. Repository structure

```
hitl-gate-kit/
  spec/                 HGK-Core, -Approver, -Policy, -Actor, -Audit, -Rollout
  schemas/              JSON Schemas + OpenAPI
  conformance/          suite source, Dockerfile, mock IdP, sample hgk.config.yaml
  examples/
    broken-service/     deliberately faulty target (suite self-test)
    express-minimal/    smallest correct integration (v0.2)
  packages/             @hgk/* (v0.2)
  guide/                implementation guide
  docs/                 PRD, decisions, architecture, design, implementation plan, tests/
  .github/              CI (actions pinned to SHA), CODEOWNERS, templates
  SECURITY.md  CONTRIBUTING.md  CHANGELOG.md  LICENSE  LICENSE-spec
```

## 10. Scalability, hosting and supply chain

- **No hosting.** The kit provisions no cloud resources; a cloud reference architecture does not apply. It runs in the adopter's system and CI.
- **Scaling the gate** is the adopter's concern, but the specification keeps it simple: tickets are short-lived rows with atomic conditional updates, so any database with compare-and-set works; no locks or extra infrastructure.
- **Suite cost:** one container, about 30 tests, minutes per run; fits a pull-request CI job.
- **Supply chain:** CI on every pull request runs format, lint, typecheck, unit tests, schema validation, the conformance self-test, secret scan, dependency review and static analysis. Third-party actions pinned to commit SHAs; changes to CI or build files need owner review. Releases: signed tags, npm provenance, CycloneDX SBOM, signed container image, SLSA build level 2 to start; always human-gated.
