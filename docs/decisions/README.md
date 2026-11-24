# NexCall decision log

This directory is the canonical record of consequential project decisions across the parent,
API, web and mobile repositories. Plans describe implementation work; handoffs describe client
contracts. Both link back here for current decision status.

## Records

- [Shared agent configuration](2026-11-22-shared-agent-configuration.md): AGENT-001 records
  requested common fields and templated prompts; AGENT-002 records failed-transfer behavior and billing
  limits; AGENT-003 implements shared catalogues/configuration and the accepted Vapi warm transfer in
  the API checkout; AGENT-004 consolidates the catalogue HTTP handlers. The final check-in review
  passed the MySQL-enabled build with 927 tests passed and one opt-in live test skipped. Live transfer behavior,
  human-leg billing evidence, frontend and deployment remain separate.
  The 2026-11-23 integration with API `master` is recorded in the same document: 939 tests passed,
  one opt-in live test skipped, using a temporary 2 GB test heap after the default-heap run exhausted memory.

| Record | Scope |
|---|---|
| [CRM and campaign baseline](2026-11-05-crm-backend.md) | Original CRM-001 through CRM-024, with replacement links for changed behavior |
| [Platform and delivery](2026-11-18-platform-and-delivery.md) | Repository boundaries, monolith, deployment, storage, identity, dates and documentation. [PLAT-009](2026-11-18-platform-and-delivery.md#plat-009-correlate-api-request-and-workflow-logs) records the implemented correlation for API requests, external calls and background work. |
| [Campaigns and calling](2026-11-18-campaigns-and-calling.md) | Task selection, dispositions, hours, attempts, retries, recordings, scripts and duplication |
| [Inbound routing and agents](2026-11-18-inbound-routing-and-agents.md) | IN-008/IN-009/IN-010 implemented; direct Twilio onboarding and API-wide preflight removal IN-011/IN-012 implemented and locally verified ([plan](../../apps/api/docs/superpowers/plans/2026-11-18-twilio-number-onboarding.md)); client integration and rollout remain separate |
| [Number associations and release](2026-11-19-number-associations-and-release.md) | NUM-001 through NUM-005 implemented in the API checkout; role-scoped associations/counts, database release to the pool, restart guards and zero-live local Vapi cleanup; frontend handoff delivered, production rollout separate |
| [Voice catalog, CRM export and campaign language](2026-11-19-voice-catalog-crm-and-language.md) | CRM-025/CALL-013/NUM-006 implemented locally; CALL-015/016 defaults and language fallbacks verified with 767 tests and bootJar; CALL-017 [API implementation and handoff complete](../../apps/api/docs/api/campaign-voice-selection-web-handoff.md), verified with 826 tests and bootJar after one review/fix pass; VOICE-002 shares private/custom voices, VOICE-003 accepts available previews, VOICE-004 records exact identities and bounded live reads; VOICE-005 implements inbound shared voices (2026-11-22); localization, frontend and deployment remain separate |
| [Subscriptions and call limits](2026-11-18-subscriptions-and-call-limits.md) | Commercial terms, settlement, usage, rollout, overrun and duration controls |

The 2026-11-18 records consolidate the dated plans, handoffs and current implementation available
in this checkout. Their filenames identify the consolidation date, not a new approval date.
Original dates and historical CRM IDs remain intact. An implementation choice is not attributed
to the user unless the source records that authorization.

## Status meanings

2026-11-24: [SUB-019](2026-11-18-subscriptions-and-call-limits.md#sub-019-tenant-subscription-cancellation)
adds current-period cancellation, immediate unpaid removal, preserved trial/payment history and
fresh assignment after completion; scheduled trial cancellation blocks base payment.

2026-11-23: [SUB-018](2026-11-18-subscriptions-and-call-limits.md#sub-018-package-trials)
adds one trial per tenant, independent AI/operator trial pools, day expiry awaiting the original
invoice, immediate fresh paid access on settlement, and migration/rollout requirements.

2026-11-23: [SUB-017](2026-11-18-subscriptions-and-call-limits.md#sub-017-package-unpublish-and-retirement)
adds unused-package unpublish, draft-only editing and preserved archival billing, with one
[consolidated subscription API handoff](../../apps/api/docs/api/subscriptions-web-handoff.md).

2026-11-23: [SUB-016](2026-11-18-subscriptions-and-call-limits.md#sub-016-current-tenant-subscription-and-access)
adds a current-tenant subscription/access read after login, with identity-derived scope and separate
subscription gates. API verification, client wiring and deployment are tracked separately.

2026-11-23: [SUB-015](2026-11-18-subscriptions-and-call-limits.md#sub-015-positive-integer-package-price-duration)
replaces fixed 1/3/12-month price terms with any positive integer month duration, retaining one-month
packages. API implementation and verification are separate from client integration and deployment.

2026-11-22: [VOICE-005](2026-11-19-voice-catalog-crm-and-language.md#voice-005-inbound-uses-the-shared-voice-catalog)
implements inbound use of the shared voice catalog and removes `/api/inbound/voices`.
The subsequent [VOICE-006](2026-11-19-voice-catalog-crm-and-language.md#voice-006-remove-inbound-legacy-voice-compatibility)
removes legacy voice compatibility by explicit user instruction.
Client integration and deployment remain separate.

| Status | Meaning |
|---|---|
| Accepted | An explicit decision or agreed requirement; implementation status is stated separately where needed |
| Implemented | Present in the inspected checkout; this does not assert deployment or a fresh test run |
| Proposed | Described in a source plan, without implementation or acceptance established |
| Deferred | Explicitly outside the current delivery or awaiting a separate decision |
| Superseded | Historical decision retained with a link to its replacement |

Approval and delivery are separate facts. Where code resolves an old proposal but no user approval
is recorded, the entry says **Implemented · Technical choice**. Do not convert that into an
accepted product decision. Historical test reports retain their original dates and counts;
documentation consolidation does not rerun those tests.

## Maintaining the record

Follow the [root repository instruction](../../AGENTS.md#decision-log). Update the relevant topic
and index in the same task that changes a decision. Include its reason, scope, consequences and
evidence. Preserve replaced choices and link both directions. Keep genuinely unresolved work
visible, including the proposed [callback-time extension](2026-11-18-campaigns-and-calling.md#call-007-optional-callback-time)
and deferred subscription features.
