# NexCall decision log

This directory is the canonical record of consequential project decisions across the parent,
API, web and mobile repositories. Plans describe implementation work; handoffs describe client
contracts. Both link back here for current decision status.

## Records

| Record | Scope |
|---|---|
| [CRM and campaign baseline](2026-11-05-crm-backend.md) | Original CRM-001 through CRM-024, with replacement links for changed behavior |
| [Platform and delivery](2026-11-18-platform-and-delivery.md) | Repository boundaries, monolith, deployment, storage, identity, dates and documentation |
| [Campaigns and calling](2026-11-18-campaigns-and-calling.md) | Task selection, dispositions, hours, attempts, retries, recordings, scripts and duplication |
| [Inbound routing and agents](2026-11-18-inbound-routing-and-agents.md) | IN-008/IN-009/IN-010 implemented; direct Twilio onboarding and API-wide preflight removal IN-011/IN-012 implemented and locally verified ([plan](../../apps/api/docs/superpowers/plans/2026-11-18-twilio-number-onboarding.md)); client integration and rollout remain separate |
| [Number associations and release](2026-11-19-number-associations-and-release.md) | NUM-001 through NUM-005 implemented in the API checkout; role-scoped associations/counts, database release to the pool, restart guards and zero-live local Vapi cleanup; frontend handoff delivered, production rollout separate |
| [Voice catalog, CRM export and campaign language](2026-11-19-voice-catalog-crm-and-language.md) | CRM-025/CALL-013/NUM-006 implemented locally; CALL-015/016 defaults and language fallbacks verified with 767 tests and bootJar; CALL-017 [API implementation and handoff complete](../../apps/api/docs/api/campaign-voice-selection-web-handoff.md), verified with 826 tests and bootJar after one review/fix pass; VOICE-002 shares private/custom voices, VOICE-003 accepts available previews, VOICE-004 records exact identities and bounded live reads; inbound migration, localization, frontend and deployment remain separate |
| [Subscriptions and call limits](2026-11-18-subscriptions-and-call-limits.md) | Commercial terms, settlement, usage, rollout, overrun and duration controls |

The 2026-11-18 records consolidate the dated plans, handoffs and current implementation available
in this checkout. Their filenames identify the consolidation date, not a new approval date.
Original dates and historical CRM IDs remain intact. An implementation choice is not attributed
to the user unless the source records that authorization.

## Status meanings

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
