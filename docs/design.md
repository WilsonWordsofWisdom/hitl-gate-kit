# HITL Gate Kit: Design

The kit is mostly a specification, a test suite and a guide, so it has few screens. This document covers the surfaces people do see:

1. the **conformance report** (HTML) and the suite's terminal output;
2. **agent-facing messages** in the gated response;
3. the **reference approval page** pattern the guide recommends to adopters;
4. **document conventions** for the specification and guide.

Baseline: Material Design 3 for layout, type, spacing and components; WCAG 2.2 AA for contrast, keyboard and screen readers. Mockups for the report and approval page are to be produced and signed off before K3 (see [implementation-plan.md](implementation-plan.md)).

## 1. Conformance report (HTML)

**Who reads it:** a tester checking what failed, and an assessor deciding whether to accept the result. Both need the verdict first, then the evidence.

**Layout, top to bottom:**

| Area | Contents |
|---|---|
| Header | Kit and suite version, target base URL, date and time, config file hash, approver driver used |
| **Verdict** | The highest HGK level achieved (or "Not conformant"), as the most prominent element on the page. Counts of pass, fail and skip |
| Level summary | One row per level (1, 2, 3): achieved or not, and which requirement IDs blocked it |
| Requirements table | One row per requirement ID: status, test names, short reason. Failures sorted first. Filter by status and by group (Core, Approver, Audit, MCP) |
| Evidence detail | Expandable per test: request and response (secrets and tokens redacted), expected vs actual, timing |
| Footer | How to reproduce: the exact `docker run` command and config used |

**Rules:**
- **Status is never colour alone.** Each status has an icon and a word: ✓ Pass, ✗ Fail, – Skip.
- **Skips are explained.** A skipped requirement says why (not configured, needs Level 3, manual driver not run) and counts against the level it belongs to.
- **Redaction** in evidence: `Authorization`, cookies, `HITL-Ticket` values after use, enrolment codes and approval assertions are replaced with `[redacted]`.
- **Self-contained:** one HTML file, no external requests, so it can be attached to an assessment.
- **States:** a run that could not start (bad config, target unreachable) produces a report that says so plainly, with the error, instead of an empty table.
- Works printed and on narrow screens; the requirements table becomes stacked cards below 600 px.

**Terminal output:** one line per test with requirement IDs, then the verdict line, e.g. `HGK Level 1: PASS · Level 2: FAIL (CORE-15) · 27 pass, 1 fail, 2 skip`. Exit code is non-zero if the level set by `--require-level` is not met.

## 2. Agent-facing messages (CORE-14)

The gated response's `message_for_user` is read by an agent and often relayed to the person. It must survive being read by a model without becoming an instruction to it.

**Rules:**
- **Declarative and addressed to the person.** "This payment needs your approval. Open the link to review and approve it." Not "Agent: stop and ask the user…".
- **No imperatives aimed at models,** no "ignore", "you must", "as an AI". The suite lints for these, using the prototype's text lint.
- **No secrets,** no ticket-redeeming tokens beyond the ticket ID, no personal data beyond what the person already sees.
- **States what happens next:** the link, how long it is valid, and that nothing happens until they approve.
- Plain language, under 300 characters; the specification includes a few example messages.

## 3. Reference approval page (APR-4, APR-6, APR-7)

The adopter builds this page on their own origin. The guide gives a reference pattern; the conformance suite tests the behaviour, not the look.

**Hierarchy:**
1. **What you are approving,** from the ticket record (never from URL parameters): action, resource, the key values (amount, recipient, what will be deleted).
2. **Confirmation code** matching the one shown where the request was made (APR-7).
3. **Who asked and from where:** "Requested by an AI agent" when there is agent evidence; time of request; time left.
4. **One primary action** ("Approve with passkey") and a clear secondary ("Decline").

**States to design:** pending (default), approving (waiting for the authenticator), approved, declined, expired, already used, wrong account, rate limited, and a "use another device" state with a QR code when APR-6 applies. Each has a plain message and the next step.

**Accessibility:** one `h1`; the summary is a description list; focus moves to the result after approval; the countdown is announced politely, not every second; passkey failure has a recovery route (the guide's accessibility and recovery chapter).

## 4. Specification and guide conventions

- **Specification:** Markdown, one file per part. Each requirement is a numbered block starting with its ID in bold (**CORE-9**), using RFC 2119 keywords in capitals, followed by a "Rationale" line and a "Tested by" line naming conformance tests.
- **Guide:** task-led chapters (see the outline in [implementation-plan.md](implementation-plan.md), K5), each opening with who it is for and what you will have at the end. Code samples in Java, C#, Python and TypeScript as pseudocode plus each framework's own interception points.
- **Status labels:** every profile and section is marked **Stable**, **Draft** or **Informative** at the top.
- **Diagrams:** Mermaid in Markdown, so they render on GitHub and stay versionable.
- **Language:** plain English, British spelling, short sentences. Terms defined once in the glossary.
