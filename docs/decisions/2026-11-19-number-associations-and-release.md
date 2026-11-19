# Number associations, release and Vapi lifecycle

**Source/decision date:** 2026-11-19 (Europe/Bucharest). **Delivery:** Implemented in the API
checkout following explicit user approval. Frontend integration, production migration, live provider
verification and deployment remain separate. Verification results are recorded below.

The user's new requirements and explicit release clarifications are recorded below. The
[API design](../../apps/api/docs/superpowers/specs/2026-11-19-twilio-number-associations-and-release-design.md)
and [implementation plan](../../apps/api/docs/superpowers/plans/2026-11-19-twilio-number-associations-and-release.md)
contain the contracts and completed work sequence. The [frontend handoff](../../apps/api/docs/api/twilio-number-associations-and-release-web-handoff.md)
is delivered in the API repository. API-only scope remains in force. See the [status definitions](README.md).

### NUM-001 Role-scoped associations and number counts

**Status:** Implemented · Explicit user requirement, approved for API-only implementation.

Use one campaigns-association endpoint for `ADMIN` and `CLIENT_ADMIN`. A client admin may see only
its own tenant campaigns/numbers; an admin may inspect any number, receives tenant identity, and
can filter the number list by tenant. Campaign details should be minimal: ID, name and status.
Expose how many campaigns are associated with each visible number.

**Implemented technical contract:** Keep `GET /api/settings/twilio-numbers`, add derived
`campaignCount`, and add `GET /api/settings/twilio-numbers/{id}/campaigns`. Include `tenantId` only
in the admin association representation and a small `campaignType` field for client navigation.
Count distinct current associations: undeleted `IDLE`/`ACTIVE` campaigns (legacy null status
conservatively included), including actual/pending inbound-agent reservations. Finished/deleted
campaigns retain history without counting as current allocations. This live-only interpretation
was part of the approved plan; its exact query/representation remains an implementation choice.

Source: the user's four-part feature request in this repository chat; plan Task 1.

### NUM-002 Immediate database release to the unassigned pool

**Status:** Implemented · Explicit user clarification, approved for API-only implementation.

Only an admin may release a number. Return it to NexCall's unassigned pool and break database
campaign/tenant associations so it can be reused. Retain ownership in Twilio. Pause its associated
live campaigns and prevent new calls; ongoing calls must be left alone and may finish.

**Implemented technical contract:** A dedicated `POST /api/settings/twilio-numbers/{id}/release`
atomically pauses/detaches current associations, revokes queued agent ownership work, clears the
inventory tenant and saved Vapi ID, and returns the affected campaign IDs. Preserve inventory,
campaign/task records, provider resources, call IDs, signed receipts and immutable caller history.
No hangup, Twilio/Vapi deletion or provider mutation occurs in this command. New incoming calls
to unassigned pool inventory reject; authenticated old-call callbacks retain their attribution.

Stale expected phone/tenant conflicts, a provider-configuration lease conflict, compatible locking
and bounded database retry are implemented technical choices. Active calls never block release. The
new-call cutoff is provider dispatch, protected by a durable operation barrier opened in final
authorization under the inventory lock. An authorized-but-unsent request cannot pass a successful
release; release returns a busy conflict while dispatch/configuration remains open. Session loss or
lease expiry cannot clear it. An abandoned operation requires positive sender termination/fencing
and safe reconciliation; no automatic timeout unblocks it. Calls already submitted may ring/connect
later and finish. This does not recall a request already sent to a provider. Normal subscription/
duration controls remain independent. These operational tradeoffs are implemented technical choices, not additional user product decisions.

Source: the user explicitly selected "Return to NexCall's unassigned pool; retain ownership in
Twilio" and "Leave the ongoing calls as they are. Deallocate the number so new calls cannot be
performed anymore" in this chat; plan Tasks 2–3.

### NUM-003 Explicit number selection before restart

**Status:** Implemented · Explicit user requirement, approved for API-only implementation.

A campaign paused by number release cannot Play/Activate until an eligible number is explicitly
associated with it. Fail with a descriptive error. A saved historical number or an unrelated edit
must not restore calling rights.

**Implemented technical contract:** Reuse the existing displaced-number marker, expose
`numberReplacementRequired` consistently, and return targeted HTTP 409 `campaignNumberReleased`
with detail: "This campaign was paused because its phone number was released. Assign a number
before starting it again." Validate tenant ownership and internal allocation before clearing the
marker; keep the campaign paused until explicit activation. The same DID can be selected again
after legitimate reassignment; the requirement is a fresh validated association, not permanent exclusion.

Source: feature request item 3; plan Task 3. Preserve the allocation rules in
[IN-009](2026-11-18-inbound-routing-and-agents.md#in-009-share-outbound-numbers-within-one-tenant).

### NUM-004 Clear the local Vapi mapping at zero live associations

**Status:** Implemented · Explicit user requirement, approved for API-only implementation.

Automatically clear the inventory's saved `vapiPhoneNumberId` when no live campaign remains.
This clears NexCall's link, not the external Vapi phone-number resource. Paused campaigns and
actual/pending inbound reservations remain live; finished/deleted history is preserved separately.

**Implemented technical design:** Use the same association policy for counts, release and cleanup.
Cover every actual delete/finish/archive/number-change writer and correct stale mappings in a new
migration. Stop eager zero-live discovery during onboarding/tenant assignment. Resolve a missing
voice-outbound mapping outside the database transaction, then save it atomically with the new
live campaign association after current ownership/allocation checks. Preserve an admin manual
choice for ambiguous provider matches through the campaign mutation. These first-use and manual
mapping interfaces are implemented technical choices. An optional admin UUID body on duplicate
and status/start preserves a manual choice after zero-live cleanup; existing bodyless calls remain supported.

This replaces mapping retention/discovery as the desired lifecycle from
[IN-011](2026-11-18-inbound-routing-and-agents.md#in-011-direct-twilio-inventory-onboarding),
while preserving its direct Twilio ownership check and unconditional webhook configuration.
The new lifecycle is implemented in this checkout. Earlier onboarding verification retains its
original date and scope; it does not independently verify release.

Source: feature request item 4 and the preceding explanation of the inventory mapping surviving
campaign deletion; plan Task 4.

### NUM-005 Delivery and verification boundary

**Status:** Implemented · Technical choices under the user's explicit API-only implementation approval.

Implementation uses the existing toolchain, customer transaction manager and module boundaries.
Counts are derived. Migration 034 adds durable operations and allocation generations/submission states,
backfills only missing caller snapshots and clears zero-live mappings. Drain old allocation/mapping/
dispatch writers before enforcing the invariant. No new secrets or environment settings are needed.
Keep existing calling/provider credentials, origins and signed callback secrets. Only the obsolete
preflight confirmation configuration identified in IN-012 can be removed after old instances drain.

Final review identified callback attribution and stale-dispatch/ownership cases. The implemented
corrections require immutable admitted SID/caller/recipient evidence for old-agent callbacks, resolve
already-admitted calls before new-owner routing, refresh locked inventory state, and protect assistant/
SIP provisioning with durable operation barriers. Release or explicit number replacement advances an
allocation generation. Each new attempt captures it; stale prepared intents cannot revive even when
the same DID is reselected. Provably unsent preparations are cancelled/requeued with zero settled usage
and unused quota removed. Submitted/unknown legacy attempts are preserved. These are technical choices
needed to uphold the approved cutoff and ongoing-call requirements.

The API contract publishes overlapping role representations with OpenAPI anyOf; admin tenantId is
required and nullable. Existing authorization denial remains HTTP 401. Optional admin manual Vapi
mapping is available on create/update/duplicate/start; provider lookup stays outside write transactions.
Ordinary DELETE cannot remove pool inventory with live legacy references. Number normalization uses
the same accepted inventory domain across association, cleanup, detach and dispatch.

Verification includes role/contract tests, disposable-MySQL release/allocation/dial races, connected-call
completion/settlement after reassignment, first-use Vapi mapping, pre-034 upgrade/fresh-schema validation,
one independent final review and regression fixes. Final wrapper/build results are recorded in the
[implementation plan](../../apps/api/docs/superpowers/plans/2026-11-19-twilio-number-associations-and-release.md).
Production migration, real provider behavior and rollout capacity remain unverified. No commits,
branches, pushes, deployments or live provider/data mutations were performed. Unrelated prior web and
subscription work is preserved.

Source: explicit final user approval in this chat; API architecture analysis, final independent review,
and plan Task 5. No unresolved business clarification remains. An abandoned operation requires positive
sender fencing/termination and provider/attempt reconciliation before audited completion; no timeout or
automatic recovery endpoint is introduced. Availability and operational recovery in production remain
rollout responsibilities, not evidence supplied by local tests.
