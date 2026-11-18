# Inbound routing and agent decisions

Consolidated on 2026-11-18 (Europe/Bucharest). These are backend implementation decisions;
frontend integration and provider rollout are separate. See the [status definitions](README.md).

The user approved backend-only implementation of the simpler policy on 2026-11-18 in IN-008
through IN-010 below. That policy is now implemented in this checkout; older entries identify
their superseded scope. The Collections sharing exception is confirmed in IN-009. The
[implementation plan](../../apps/api/docs/superpowers/plans/2026-11-18-inbound-routing-simplification.md)
tracks delivery and verification. Web/mobile integration and production rollout remain separate.

### IN-001 Operator inbound mode and matched history

**Source date:** 2026-11-13. **Status:** Partially superseded by [IN-008](#in-008-preserve-collections-inbound-and-reject-other-outbound-returns) and [IN-010](#in-010-simplification-implementation); Collections behavior remains implemented.

Originally, `acceptInboundCall` enabled live operator routing across outbound types. New campaigns
defaulted false; the migration enabled existing live operator campaigns. Explicit true with
`VOICE_AGENT` returned `invalidCampaignInboundMode`. IN-008/IN-010 restrict this flag to Collections
and normalize unsupported values to false instead of that validation failure.
Match verified inbound calls to legitimate outbound history using caller, dialed number and
tenant. Retain one history row per trusted Twilio CallSid, including calls logged without live
routing. Dedicated AI inbound agents use `AGENT_INBOUND` and their own endpoints.

Sources: [callback plan](../../apps/api/docs/superpowers/plans/2026-11-13-inbound-campaign-callbacks.md),
[campaign validation](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CampaignService.java),
[inbound handoff](../../apps/api/docs/api/inbound-campaign-callbacks-web-handoff.md).

### IN-002 Durable missed-call queue

**Source date:** 2026-11-13. **Status:** Implemented for Collections; broader outbound applicability superseded by [IN-008](#in-008-preserve-collections-inbound-and-reject-other-outbound-returns).

Keep one pending inbound callback per open task, ordered by first receipt. Due scheduled
appointments precede eligible inbound callbacks, then ordinary work. Paused/out-of-hours
campaigns can accrue pending work; finished campaigns and completed tasks retain history without
reopening. Reading Next Call or reserving an attempt does not consume the callback. Remove only
the covered entry after a live answer or provider-confirmed return dial, preserving newer calls.
Original applicability covered both execution modes. IN-008 now limits missed-inbound queue
scheduling to Collections; retained old Cold Calling rows are ignored. Ordinary scheduled
outbound callbacks and authenticated completion of historical calls remain supported.

Sources: [callback contract](../../apps/api/docs/api/inbound-campaign-callbacks-web-handoff.md),
[queue implementation](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CampaignCallbackQueue.java).

### IN-003 Dedicated inbound agents and asynchronous provisioning

**Source dates:** existing agent contract and 2026-11-17 routing plan.
**Status:** Implemented · API architecture choice; original agent approval date not established here.

Create an idle `AGENT_INBOUND` campaign with a saved Vapi assistant, structured output and trusted
SIP target. Provision outside the request through durable, revisioned worker state. Expose
effective `status` separately from `activeRequested`; a successful save can still be pending.
Null schedule means 24/7 while active; configured windows use an IANA zone, inclusive opening
and exclusive closing. New agents default recording off. Archive idle agents asynchronously,
retain their interaction history, and preserve existing task/CRM attribution.

Sources: [agent API guide](../../apps/api/docs/inbound-agents-api.md),
[agent service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/InboundAgentService.java),
[web handoff](../../apps/api/docs/api/inbound-agent-web-handoff.md).

### IN-004 Number eligibility and exclusive active use

**Status:** Active-outbound exclusivity superseded by [IN-009](#in-009-share-outbound-numbers-within-one-tenant); inventory administration remains implemented.

Previously, inventory-backed outbound numbers could remain referenced by several campaign records,
but activation rejected another ACTIVE campaign using the same number. IDLE/FINISHED outbound
campaigns did not prevent activation. IN-009 permits simultaneous active outbound sharing and
requires tenant-owned inventory; finished campaigns are excluded from live assignment lookup.
Dedicated inbound-agent reservations block normal selection even while paused. Tenant ownership
and voice-provider mapping eligibility still apply; reusing an old campaign reference does not
grant another tenant access.

Platform admins manage inventory; other users see numbers within active tenant memberships.
Assignment changes send the expected prior tenant so stale edits return a conflict. Assigned
numbers block inventory/tenant deletion, and inventory edits do not silently rewrite existing
campaigns or live calls.

Sources: [number service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/TwilioNumberService.java),
[assignment query](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/repositories/CampaignRepository.java),
[reservation query](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/repositories/TwilioNumberRepository.java),
[tenant inventory contract](../../apps/api/docs/tenant-settings.md).

### IN-005 Confirm takeovers and external route replacement

**Source dates:** agent contract and 2026-11-17. **Status:** Campaign takeovers superseded by [IN-009](#in-009-share-outbound-numbers-within-one-tenant)/[IN-010](#in-010-simplification-implementation); external-route confirmation remains implemented.

Preflight reports affected campaigns and provider routing conflicts. Originally a campaign
takeover used an override flag and five-minute token, then paused and detached the incumbent at
cutover. IN-009/IN-010 replace this with hard allocation conflicts and fence legacy takeover jobs.
Replacing an external Twilio route on an otherwise eligible number still requires the override
flag, five-minute token and a platform admin.
Reject new inbound calls during cutover while connected calls can finish. Refresh preflight when
confirmation becomes stale. Verify number ownership/voice capability and keep durable setup,
retry and restoration state. Ordinary setup preserves unapproved external routes.

Sources: [preflight and override handoff](../../apps/api/docs/api/inbound-agent-web-handoff.md),
[shared routing plan](../../apps/api/docs/superpowers/plans/2026-11-17-shared-inbound-voice-webhook.md).

### IN-006 One shared signed Voice webhook

**Source date:** 2026-11-17. **Status:** Implemented · Routing architecture choice.

Route NexCall-managed numbers through signed POST `/api/twilio/incomingCall`. Resolve committed
ownership locally; ready active inbound agents within hours bridge directly to authenticated
Vapi SIP. Paused, unready, ambiguous or closed agent routes reject before answer. Numbers with
no effective agent assignment or reservation use the Collections operator/callback path. Idle or
pending agent reservations reject rather than falling back to old outbound history. Keep provider provisioning
out of the live routing request. Historical ownership correlates late status/report callbacks.
Call setup now depends on NexCall and its database; connected SIP calls can continue during an
API outage. A static reject-only Twilio fallback is optional.

Sources: [shared webhook plan](../../apps/api/docs/superpowers/plans/2026-11-17-shared-inbound-voice-webhook.md),
[routing implementation](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/InboundVoiceRouting.java).

### IN-007 Provisioning updates after commit

**Source date:** 2026-11-18. **Status:** Implemented · Backend notification choice.

Publish safe immutable provisioning snapshots after commit to authorized sessions subscribed to
`/user/task-events/inboundAgentUpdates`. Platform admins receive all tenants; client admins only
their active memberships. Reject client publishing and raw broker subscriptions. Coalesce writes
within one transaction while retaining distinct committed stages and per-session delivery order.
Send no credentials or scripts. Events are best-effort refresh hints; subscribe then load REST,
and reload after reconnect. The process-local broker supports the current single-replica setup;
shared transport/fan-out is required before relying on this channel across replicas.

Sources: [completed WebSocket plan](../../apps/api/docs/superpowers/plans/2026-11-18-inbound-provisioning-websockets.md),
[event contract](../../apps/api/docs/inbound-agents-api.md#provisioning-updates-over-websocket),
[delivery service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/InboundAgentUpdateDelivery.java).

### IN-008 Preserve Collections inbound and reject other outbound returns

**Decision date:** 2026-11-18. **Status:** Accepted · Implemented in backend checkout; web/mobile integration and deployment excluded.

Keep Collections (`DEBT_RECOVERY`) inbound behavior unchanged. For other outbound campaign types,
reject incoming calls instead of entering their live inbound/callback workflow. Dedicated
`AGENT_INBOUND` campaigns retain their inbound-agent behavior. The target narrows IN-001 and
IN-002; it does not authorize changing Collections routing, availability or disposition rules.

Source: the user's inbound simplification request and subsequent backend-only plan approval in
this chat. NexCall returns rejection through the shared signed webhook, as recorded in IN-010.
A shared Collections/Cold Calling number still uses campaign attribution before applying the rule.

### IN-009 Share outbound numbers within one tenant

**Decision date:** 2026-11-18. **Status:** Accepted · Implemented in backend checkout; web/mobile integration and deployment excluded.

A number belongs to one tenant and may be allocated to multiple outbound campaigns regardless
of outbound type. Remove the current restriction against multiple active outbound campaigns
sharing it. Keep dedicated inbound-agent use separate from outbound use. This supersedes active
outbound exclusivity in IN-004 and campaign takeover permission in IN-005. IDLE/ACTIVE outbound
allocations block agent allocation; FINISHED/deleted historical references alone do not. Unfinished
or unresolved calls must finish before conversion. Tenant ownership and provider mapping still apply.

**Confirmed Collections exception:** Collections may receive return calls on numbers shared by
outbound campaigns. An `AGENT_INBOUND` campaign's number stays exclusive to inbound-agent use
and cannot also be allocated to Collections or any other outbound campaign. The user explicitly
confirmed this interpretation in the follow-up answer. Actual and pending agent reservations
remain exclusive while idle, pending or active. Existing historical campaign associations remain intact.

Source: the user's number-allocation requirements and explicit clarification in this chat.

### IN-010 Simplification implementation

**Source date:** 2026-11-18. **Status:** Accepted · Implemented in backend checkout; final verification recorded in the implementation plan.

Keep the shared signed Voice webhook, rejecting non-Collections outbound matches
before new inbound workflow mutations, allowing tenant-owned outbound number sharing and making
inbound-agent allocation conflicts hard conflicts instead of automatic incumbent takeovers.
Twilio-hosted static rejection remains an alternative for numbers that never serve Collections.
The implementation ignores old cold missed-inbound queue entries, preserves connected-call
completion, and retains external-route override approval separately from campaign allocation.
Normalize non-Collections inbound flags to false. Serialize allocation on tenant/inventory locks
and use current committed reservation reads. Keep old history and allow a finished campaign to
move away from its now-reserved historical number. Reject malformed legacy numbers on activation.
Fence both queued initial takeovers and legacy replacement takeovers before provider mutation;
retain guarded recovery for a cutover already started and archive restoration.

The user approved this plan for backend-only execution on 2026-11-18. Web/mobile changes are
excluded, with client contract differences documented in the backend handoff. The
[separate implementation plan](../../apps/api/docs/superpowers/plans/2026-11-18-inbound-routing-simplification.md)
contains task order, affected files, tests, compatibility and rollout checks. The
[contract delta](../../apps/api/docs/api/inbound-routing-simplification-web-handoff.md) describes
client follow-up. A [read-only audit](../../apps/api/docs/sql/inbound-number-allocation-audit.sql)
is prepared for rollout. Production data/provider checks and deployment have not been performed.

**Verification, 2026-11-18:** final backend `./gradlew --offline test bootJar` passed 641 tests
with zero failures, errors or skips, including 10 MySQL allocation/concurrency cases. All five
audit SELECTs executed on a disposable migrated schema. Providers were mocked; no client
build/test or production/provider pilot is asserted.
