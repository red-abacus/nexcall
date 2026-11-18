# Campaign and calling decisions

Consolidated on 2026-11-18 (Europe/Bucharest). These entries record API behavior unless web or
mobile delivery is explicitly stated. Status meanings are in the [index](README.md).

### CALL-001 Automatic next task

**Source date:** 2026-11-07. **Status:** Implemented · Explicit user request recorded in the plan.

Web and mobile may call `GET /api/operator/callTask/next` without parameters. The server searches
assigned active campaigns, preserving Collections continuation and Cold Calling eligibility.
`campaignType` remains an optional filter. The server CRM flag controls whether cold/linked tasks
are eligible; no client `crmCapable` parameter is required. This supersedes
[CRM-018](2026-11-05-crm-backend.md#crm-018--keep-old-clients-eligible-for-compatible-tasks).

Sources: [automatic next-call plan](../../apps/api/docs/superpowers/plans/2026-11-07-automatic-next-call.md),
[controller](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/controllers/OperatorCallTaskController.java).

### CALL-002 Disposition identity and CRM history

**Source dates:** 2026-11-05 through 2026-11-13. **Status:** Implemented · Evolved API contract.

Preserve attempt-level identity, campaign access and operator ownership. Keyed outbound
dispositions use their verified attempt; answered inbound dispositions identify their own
`callDetailsId`. Narrow legacy unlinked feedback compatibility remains. Trusted voice-agent
results and matched inbound events can also contribute CRM history, so history is no longer
restricted to keyed outbound human dispositions. This broadens historical CRM-006 and
[CRM-017](2026-11-05-crm-backend.md#crm-017--limit-linked-history-to-verified-outbound-attempts);
it does not remove the transactional outbox or tenant checks.

Sources: [feedback service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CallDetailsService.java),
[inbound feedback contract](../../apps/api/docs/api/inbound-campaign-callbacks-web-handoff.md),
[voice disposition contract](../../apps/api/docs/api/vapi-voice-agent-handoff.md).

### CALL-003 Done and date-only reschedule

**Source date:** 2026-11-10. **Status:** Implemented · Product behavior recorded in the handoff.

Cold Calling supports `DONE`, `RESCHEDULE` and optional `HOT`/`COLD` qualification. Require a
confirmed answered attempt for Done/Reschedule, preserve provider status separately, and treat
an identical saved outcome as a replay. Reschedule keeps the same task, validates the callback
date against the campaign and returns the task to the queue. Future callbacks are excluded;
eligible callbacks due today precede overdue callbacks and ordinary work. Collections keeps its
existing outcomes.

Sources: [operator disposition](../../apps/api/docs/api/cold-calling-disposition-handoff.md),
[Vapi disposition plan](../../apps/api/docs/superpowers/plans/2026-11-10-vapi-cold-call-disposition.md).

### CALL-004 NexCall owns voice scheduling

**Source date:** 2026-11-10. **Status:** Implemented · Agreed behavior in the source plan.

NexCall reserves and rechecks due work, then submits individual immediate Vapi calls outside the
database transaction. Keep callbacks, retries and hours in NexCall; do not send future
`schedulePlan` requests for new campaign calls. Retain uncertain submissions for reconciliation
before creating replacements. Preserve the existing one-active-voice-attempt-per-tenant guard.
A provider-accepted request can ring after pause or closing; a pause blocks subsequent submissions.

Sources: [immediate-call plan](../../apps/api/docs/superpowers/plans/2026-11-10-immediate-vapi-campaign-calls.md),
[voice worker](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/VoiceCallWorker.java).

### CALL-005 Campaign working hours and live operator counts

**Source date:** 2026-11-10. **Status:** Implemented · Shared policy.

Use campaign-level IANA time zones, selected weekdays and one same-day window. Opening is
inclusive and closing exclusive; overnight windows are unsupported. No saved hours means no
hours restriction within campaign dates, with UTC for unscheduled voice campaigns and the
existing operator fallback. Preserve compatible nested voice hours and reject conflicting inputs.
Recheck before dialing without interrupting connected calls. Today's `remainingTasks` counts
each currently dialable task once using the same eligibility as Next Call; past counts remain
saved snapshots.

Sources: [hours plan](../../apps/api/docs/superpowers/plans/2026-11-10-operator-campaign-working-hours.md),
[count plan](../../apps/api/docs/superpowers/plans/2026-11-10-operator-dialable-task-count.md),
[hours policy](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CampaignHoursPolicy.java).

### CALL-006 Campaign attempt limits and retry spacing

**Source date:** 2026-11-10. **Status:** Implemented · Replaces the initial tenant-wide policy.

Save total/daily attempt caps and voice retry delay per campaign. Count authorization once per
campaign and normalized destination across Twilio direct, conference and Vapi calls. Shared
numbers within one campaign share usage; other campaigns have separate budgets. Daily reset is
midnight UTC. Platform values seed new campaigns; a duplicate copies policy with fresh usage.
The daily cap may equal the total cap on a campaign. Default voice spacing is 60 minutes from
the last attempt's end, across numbers and midnight. Preserve answered outcomes; close an
exhausted task with its actual outcome and `TOTAL_ATTEMPTS_REACHED`.

The original [platform-settings plan](../../apps/api/docs/superpowers/plans/2026-11-10-outbound-call-settings.md)
is superseded for tenant-wide counting and applying later platform edits to existing campaigns.
Its initial defaults are six total/two daily. Do not reconstruct charges from arbitrary old call
history; retain the durable authorization ledger and its explicit migration/backfill rules.

Sources: [campaign policy plan](../../apps/api/docs/superpowers/plans/2026-11-10-campaign-attempt-limits-and-retries.md),
[current policy contract](../../apps/api/docs/api/campaign-outbound-policy-web-handoff.md),
[quota implementation](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/OutboundCallQuota.java).

### CALL-007 Optional callback time

**Source date:** 2026-11-10. **Status:** Proposed · Not implemented in the inspected API.

The plan proposes optional `rescheduleTime`/`callbackTime`, strict `HH:mm`, campaign-local DST
resolution, and exact-time eligibility shared by operators and Vapi. Current DTOs and services
still use callback dates and do not expose those time fields. Do not present this as delivered or
change the date-only contract during documentation maintenance. Existing explicit CRM follow-up
timestamps and automatic retry times are separate features.

Source: [callback-time plan](../../apps/api/docs/superpowers/plans/2026-11-10-callback-times.md).

### CALL-008 Voice qualification and unavailable recipients

**Source dates:** 2026-11-10 and 2026-11-11. **Status:** Implemented · Provider-analysis policy.

Use saved structured-output contracts to distinguish legacy opt-out results from cold-call
dispositions. Apply qualification and confirmed callback dates only to supported conversation
evidence; invalid/missing analysis is bounded and can require review. Use Vapi voicemail
detection and the normal `endCall` tool. Consistent `VOICEMAIL`/`NO_CONVERSATION` unreached results
can produce `NO_ANSWER` and retries; contradictory reached-recipient evidence requires review.
Retain the provider end reason and distinguish automatic `retryAt` from an agreed callback date.

Sources: [voicemail plan](../../apps/api/docs/superpowers/plans/2026-11-11-vapi-voicemail-and-retries.md),
[current provider contract](../../apps/api/docs/api/vapi-voice-agent-handoff.md),
[analysis implementation](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/VoiceCallAttempts.java).

### CALL-009 Voice recording defaults

**Source date:** 2026-11-10. **Status:** Implemented · Testing default replaces the original opt-in design.

For Voice Agent Cold Calling, an explicit saved boolean controls future attempts. An omitted
choice, including on a duplicate, uses the environment default, currently true in configuration.
Switching to Operator clears the saved voice choice. Existing recordings remain available under
provider retention. The original false-by-default and duplicate-off proposals are superseded;
do not describe the testing default as a permanent production approval.

Sources: [historical recording plan](../../apps/api/docs/superpowers/plans/2026-11-10-campaign-vapi-recording.md),
[current recording contract](../../apps/api/docs/api/campaign-outbound-policy-web-handoff.md#voice-agent-recording-setting),
[configuration](../../apps/api/src/main/resources/application.properties).

### CALL-010 Shared attempt recording playback

**Source date:** 2026-11-12. **Status:** Implemented · Shared provider contract.

Keep media at Twilio/Vapi. Authorize the exact attempt and issue a fresh no-store playback link.
`hasRecording` reflects saved provider confirmation, not merely a recording request; history
reads do not call providers. Reconcile delayed artifacts separately. Twilio playback uses a
short-lived API media grant; Vapi supplies an allowed signed URL. This replaces the original
Vapi-only/no-backfill scope while preserving compatible legacy fields.

Sources: [playback plan](../../apps/api/docs/superpowers/plans/2026-11-12-shared-recording-playback.md),
[playback handoff](../../apps/api/docs/api/shared-recording-playback-web-handoff.md),
[playback service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/voice/VoiceRecordingPlayback.java).

### CALL-011 Shared guide and reusable script snapshots

**Source date:** 2026-11-10; preset contract date not established here. **Status:** Implemented.

Cold Calling stores one opening and guidance in `callGuide` for human and voice execution, with
task placeholder rendering and legacy voice/script compatibility. Reusable presets live under
`/api/settings/scripts`; applying one copies its text into a campaign. Editing/deleting the
preset does not rewrite prior campaigns. Collections retains its RPC/MESSAGE assignment contract.
This extends the original CALL-script design in CRM-005.

Sources: [shared guide plan](../../apps/api/docs/superpowers/plans/2026-11-10-shared-cold-calling-guide.md),
[preset contract](../../apps/api/docs/api/settings-scripts-web-handoff.md).

### CALL-012 Duplicate into a separate paused campaign

**Status:** Implemented · API contract; original decision date not established here.

Duplicate ACTIVE, IDLE or FINISHED campaigns with a new campaign ID in IDLE, leaving the source
unchanged. Copy eligible task data, active links and configuration, with new idle task IDs and
fresh attempts, feedback, callbacks and statistics. Exclude deleted tasks, archived contacts and
generated follow-ups. Each successful request creates another copy; do not automatically retry
an uncertain response. Reuse a number only when current ownership/reservation rules permit it.

Sources: [duplication contract](../../apps/api/docs/api/campaign-replay-web-handoff.md),
[campaign service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CampaignService.java).
