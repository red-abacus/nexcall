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

### CALL-018 Allow Vapi recording links hosted on Cloudflare R2

**Source date:** 2026-11-25. **Status:** Implemented in the local API checkout · Technical choice.

The user reported 503 voiceAgent.recordingProviderUnavailable for inbound interaction playback,
including agent 70 / interaction 27. Read-only production lookup identified its Vapi call. A private
provider GET returned 302 with a signed standard Cloudflare R2 bucket/account hostname, while the
call GET returned 200 and retained recording artifacts. A one-byte media range probe returned
206 audio/wav, proving that this sample recording exists and is accessible. No call was placed.

CALL-010's shared Vapi playback path accepted only built-in S3/GCS or explicitly configured hosts;
production has no Vapi recording-storage override. Thus a valid R2 redirect was rejected locally
and translated into 503. Extend the built-in host validation to standard R2 account endpoints
(32 hexadecimal account ID), optionally prefixed by a bucket label. Preserve HTTPS, no userinfo,
no fragment, default/443 port, existing providers/custom hosts and no server-side redirect following.
Do not expose provider credentials or persist signed URLs. Both inbound and outbound Vapi playback
use this adapter; the client playback contract is unchanged. No new recordings or database migration
are needed to make retained recordings eligible for playback after release.

Sources: user report and inspected VapiClient / VoiceRecordingPlayback; sanitized provider response
metadata and byte-range probe; [Vapi artifact retrieval](https://docs.vapi.ai/assistants/retrieve-call-artifacts),
[Cloudflare R2 endpoints](https://developers.cloudflare.com/r2/api/tokens/).
Configuration/client reference: [Vapi handoff](../../apps/api/docs/api/vapi-voice-agent-handoff.md#provisioning-and-staging-checks).

Local regression checks cover signed bucket/account R2 redirects and rejection of insecure schemes,
lookalike suffixes, userinfo, custom ports, fragments and malformed account hosts. Commit, push,
deployment and production playback-endpoint verification remain separate from this local correction.

Verification, 2026-11-25: all 31 selected recording and Vapi transport tests passed with zero
failures/errors/skips; bootJar passed. Diff whitespace checks passed. The earlier whole-API build
belongs to AGENT-011; it was not repeated or presented as validation of this subsequent one-line
recording-host correction. No credentials, signed links, audio files or production writes were retained.


### CALL-019 Inbound campaign recording uses the original Twilio call

**Decision/source date:** 2026-11-25. **Origin:** Explicit user requirement: record the complete
inbound conversation using Twilio when campaign recording is enabled; save no new audio when disabled.
**Status:** Published to API master at `cc7db61`. Production verification is tracked below.

Extend CALL-010 for new inbound calls. Snapshot the applied recordCalls flag at original-call
admission; replay cannot adopt a later campaign flag. Start call-level Twilio recording before
Vapi SIP, capturing both caller audio tracks through the existing parent-call transfer redirect.
Disable Vapi audio recording on all newly provisioned inbound profiles in both flag states. Keep
transcripts, summaries and collected information unchanged. Disabled calls emit no recording start
and reject recording callbacks; no Vapi fallback is allowed for modern false/pending snapshots.
Existing audio remains available, including the CALL-018 R2 fix for retained Vapi recordings.
Configuration changes affect new calls after provisioning reaches READY; ongoing calls retain
their admission-time choice. Snapshot timing and nullable legacy provenance are technical choices.

Migration 043 adds nullable policy, RecordingSid and status to the route receipt, allowing callback
arrival before interaction creation. A signed/account-verified callback must match an admitted,
allowed, recording-enabled parent call in the same campaign. Store IDs/status, never provider media
URLs. Duplicates do not emit new hints; completed metadata cannot be replaced or downgraded. List
availability and playback use the same call metadata. Reuse existing authorized recording endpoints,
Twilio grant/proxy and after-commit scoped socket hints; return one recording on the same interaction.
Existing inbound reconciliation recovers missing callbacks within its two-hour carrier window with
at most five recording-list reads per tick; no client-side provider calls or indefinite backfill.
Provider account/call identity must match, and multiple ambiguous recordings are not selected.

This supersedes the initial AI-only inbound recording limitation retained in AGENT-008/010/011.
It does not change outbound capture, interaction duration fields, AI billing, or retention/deletion
policy. Recording duration can exceed the reported AI duration. The caller recording excludes the
operator's private acceptance screening. Twilio recording/storage charges apply to the captured
duration; subscription AI minute measurement is retained.

Rollout requires migration 043, existing recording media credentials/signing key, the signed callback
route and reprovisioning every existing inbound Vapi profile to turn off its old audio capture.
The checkout does not modify remote assistants. Validate whole-call audio continuity on a live
successful transfer, plus disabled/no-transfer/refusal/no-answer and legacy playback after deployment.
A generated Start/Recording instruction is not proof of continuous audio through redirect.

Sources: explicit user direction in this conversation; inspected original caller routing/transfer;
[Twilio Start/Recording](https://www.twilio.com/docs/voice/twiml/recording),
[recording resource](https://www.twilio.com/docs/voice/api/recording).
[Implementation plan](../../apps/api/docs/superpowers/plans/2026-11-12-shared-recording-playback.md#inbound-whole-call-recording--2026-11-25),
[client handoff](../../apps/api/docs/api/inbound-agent-web-handoff.md#interactions-playback-and-crm),
[rollout guide](../../apps/api/docs/deployment-gke.md#whole-call-inbound-recording-via-twilio).

Read-only production preflight, 2026-11-25: Twilio recording API credentials are present; the
NEXCALL_RECORDING_PLAYBACK_SIGNING_KEY environment value is missing or below the existing 32-byte
minimum. Configure a dedicated shared secret before release playback testing. No key value was
printed and no production Secret or environment was modified. Inbound playback resolves the media
path against the configured public API origin, preserving the absolute playbackUrl expected by
existing inbound clients; request Host headers cannot redirect that URL.

Verification, 2026-11-25: MySQL-enabled full API build passed with 1,373 tests total,
1,372 passed, zero failures/errors, 300 MySQL tests and one optional live-provider test skipped.
After the final absolute-URL compatibility adjustment and recovery-budget tests, all 256 selected
inbound/recording/CRM tests passed (114 MySQL), zero failures/errors/skips; bootJar passed again.
The checks include signed/tampered/account-mismatched recording callbacks, admission policy replay,
enabled/disabled TwiML and Vapi provisioning, completed/pending/disabled/legacy playback, idempotence,
call/campaign scope, before-interaction callback arrival and a shared five-read recovery budget.
Liquibase XML/includes, local link targets and diff whitespace checks pass. The external 2 GB test
heap was reused; no project dependency/toolchain/heap change was introduced. Source remains local;
full-call continuity and disabled capture in live providers remain post-rollout validation.

Production configuration decision, 2026-11-25 (explicit user approval): provision a dedicated
playback signing key shared by the API replicas. Both production deployment workflows apply the
same Kubernetes application Secret, so both require the same GitHub Actions secret. The key was
configured in both repositories and the cluster. Deployment health, runtime key availability, and
playback of audio across a transfer are separate verification steps. Rotation of the five-call
reconciliation batch is deferred at the user's request.

Production verification for API `cc7db61`: [deployment and smoke test](https://github.com/red-abacus/nexcall-api/actions/runs/45636474466)
passed, migrations 042 and 043 completed, readiness reported UP, and the running pod loaded a
signing key of valid length. The unsigned recording callback returned 401 and an invalid playback
grant returned 403. These checks cover deployment and access controls. End-to-end call audio still
requires a live recorded transfer.

### CALL-020 Inbound interaction duration follows the original caller

**Source date and origin** — 2026-11-25, user report for agent 70, interaction 28.
The user verified that one recording included the AI and human conversation. Twilio reported
88 seconds for the original call and recording. The interaction displayed 27 seconds from the AI leg.
This verifies full-call playback for this transfer.

**Status** — Implemented locally. This supersedes CALL-019’s earlier duration behavior for the
top-level inbound interaction. Use Twilio’s terminal original-call duration from the existing
callback or reconciliation, independent of recording and transfer outcome. Keep the existing
provider duration as a fallback where carrier evidence is missing. The nested attempt and
subscription metering keep their existing meanings. Historical calls outside the reconciliation
window require carrier evidence to correct their display.

[Inbound client handoff](../../apps/api/docs/api/inbound-agent-web-handoff.md#interactions-playback-and-crm).

[SUB-023](2026-11-18-subscriptions-and-call-limits.md#sub-023-agent-started-calls-use-ai-minutes-through-caller-hang-up) resolves the human-leg billing decision for newly admitted agent calls.

For CALL-020, verified Twilio Call resource duration also supersedes a provisional callback
duration in the interaction list. History retains the latest accepted provider version for replay protection.

### CALL-021 Expose recording availability on outbound campaign task rows

**Source and date** — User report and production investigation, 2026-11-25. The user prioritizes future outbound calls.
The task list requires a flag on each phone entry. The API exposed availability on attempt history
but omitted it from the task list, and confirmation alone did not trigger its socket refresh.

**Status** — Pushed to API master at `84c96ce` and web master at `f9f9e86`. The local build and 111 focused API tests and 21 focused web tests passed. The user verified playback on campaign 72, attempt 236. The Vapi recording ended at handover, prompting CALL-022. Derive each phone entry’s `recordingAvailable` from the latest matching
attempt using the same rule as playback. Publish the existing task update on recording confirmation.
The campaign Listen action opens recorded playback for a completed attempt and retains live monitoring for an active call SID. Use the existing bounded Vapi checks and CALL-018 storage validation. Historical attempts were left untouched.
[Client contract](../../apps/api/docs/api/shared-recording-playback-web-handoff.md).

### CALL-022 Record the original outbound caller leg in Twilio

**Date and origin** — 2026-11-25, explicit user decision following campaign 72 attempt 236. Playback worked but the Vapi recording omitted the conversation after handover.

**Status** — Pushed to API master in `4e0cd08`. Production acceptance remains open. This supersedes the Vapi capture choice for new outbound voice-agent submissions in CALL-021.

At submission authorization, snapshot the campaign recording choice. Ask Vapi to place the call with Twilio transport recording set to that choice and Vapi audio disabled. For enabled attempts, publish the completed recording for the original Twilio call through the existing campaign and CRM playback endpoints. The initial implementation preserved old Vapi playback, superseded by CALL-023. Migration 046 adds the capture choice and Twilio identifiers. This requires a Vapi phone number backed by the tenant deployment’s own Twilio credentials.

After deployment, verify that a caller and operator can hear each other after transfer and both conversations play in one recording through final hang-up. Verify an opted-out call has no recording.

[Campaign contract](../../apps/api/docs/api/campaign-outbound-policy-web-handoff.md#voice-agent-recording-setting), [Vapi Twilio transport settings](https://docs.vapi.ai/phone-numbers/import-twilio/), [Twilio recording resource](https://www.twilio.com/docs/voice/api/recording).

### CALL-023 Serve only Twilio call recordings

**Date and origin** — 2026-11-25, explicit user decision. Historical Vapi playback is unnecessary.

**Status** — Pushed to API master in `4e0cd08`. The MySQL-enabled build passed with 1,380 tests passing and one optional live voice-preview test skipped. Supersedes the historical Vapi playback provisions in CALL-018, CALL-019, CALL-021 and CALL-022. Campaign, CRM and inbound history expose playback only for confirmed Twilio recordings. Legacy Vapi flags and artifact URLs no longer make a recording available, and background workers skip Vapi-only attempts. Call history and provider media are retained. This changes playback eligibility without a data purge.

[Client contract](../../apps/api/docs/api/shared-recording-playback-web-handoff.md).

### CALL-024 Outbound summaries follow the selected campaign language

**Date and origin** — 2026-11-25, explicit user request after Romanian campaigns returned English summaries.

**Status** — Pushed to API master in `440152b`. The full MySQL-enabled build passed with 1,382 tests passing and one optional live voice-preview test skipped. Live summary language verification is pending. When reserving an outbound voice-agent attempt, NexCall sets the Vapi post-call summary instructions to the campaign language and supplies the transcript. The existing end-report and reconciliation flows save the result in the attempt’s feedback. Previously saved summaries keep their original text.

[Outbound voice-agent contract](../../apps/api/docs/api/vapi-voice-agent-handoff.md).
