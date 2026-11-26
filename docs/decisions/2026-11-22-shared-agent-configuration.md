# Shared AI agent configuration

### AGENT-001 Common configuration and templated prompts

**Source/decision date:** 2026-11-22, Europe/Bucharest.
**Origin:** Explicit user requirements in this conversation.
**Status:** Requirements accepted, implementation authorized and present in the API checkout under AGENT-003.

Inbound and outbound AI agents should share language, industry, purpose, first message, agent
instructions, human handover conditions, transfer number, voice selection, information to collect
and running schedule. Both must support human transfer. Customer instructions extend predefined
industry/purpose/language prompt templates; the attached Romanian outbound-sales prompt is a reference.
Reuse shared voice catalogue work under VOICE-005/006. The user has requested a plan, not execution.

Proposed technical approach: common metadata/template catalogue, campaign-owned agent configurations,
one prompt/assistant composer, versioned configuration snapshots and separate direction-specific
execution. Catalogue definitions are not shared mutable agent instances. Preserve tenant isolation,
number ownership, lifecycle, provider callbacks, CRM and billing boundaries. Proposed APIs, schedule
shape, template storage and migration strategy are not yet accepted implementation decisions.

Follow-up clarification, 2026-11-22 (explicit user requirements): human handover is mandatory for both
inbound and outbound. Preserve existing schedules rather than introducing a new unified schedule
format. Information to collect is optional; concatenate its instructions to the composed prompt only
when supplied. Template catalogue and shared assistant construction direction are confirmed. The user
questioned the aggregate options endpoint and keeping voices outside the agent namespace; the revised
plan proposes separate catalogue endpoints under `/api/agents`. Exact URL relocation/cutover remains
a proposal, not an implemented or implicitly approved breaking change.

Subsequent explicit confirmations, 2026-11-22: shared catalogues use `/api/agents/...`, while inbound
and outbound retain their own creation/update endpoints and share configuration/prompt-building logic.
Templates are precreated and managed on the backend, automatically selected by industry, purpose and
language with direction-specific composition. No `/api/agents/templates` endpoint, client template
picker or full prompt-template exposure is required. Supported option combinations remain available
to clients. Template version/configuration snapshots preserve existing agents when templates change.
This replaces the earlier proposed public template endpoint; implementation remains pending.

Historical planning open points (subsequently resolved or explicitly bounded by AGENT-002/003): activation-time handover validation timing, unreachable-human behavior, direct versus warm
transfer, human availability hours, initial template coverage, outbound default-voice migration and
whether a standalone reusable agent library is desired. Existing SUB-005 does not decide charging
external human legs after AI departure. Do not infer that policy from adding transfer tools.

Sources: user's shared-fields and templating requests in this conversation; Romanian prompt attachment
`Pasted text.txt` (customer-provided reference); [implementation proposal](../../apps/api/docs/superpowers/plans/2026-11-22-shared-agent-configuration.md),
[script snapshots](2026-11-18-campaigns-and-calling.md#call-011-shared-guide-and-reusable-script-snapshots),
[billing scope](2026-11-18-subscriptions-and-call-limits.md#sub-005-provider-backed-second-accounting).

Verification/deployment: documentation only; no runtime behavior changed, tests rerun or deployment performed.

The preceding paragraph describes the initial planning-only task. Subsequent implementation and
verification are recorded under AGENT-003; it is not a current claim that runtime work is absent.

### AGENT-002 Failed human transfer and billing boundary

**Source/decision date:** 2026-11-22. **Origin:** Explicit user answer and billing simplification question.
**Status:** Failed-transfer behavior accepted. The user subsequently selected Vapi assistant-based
warm transfer and authorized implementation, with real calls tested afterward (AGENT-003).
Post-transfer human-leg billing remains unverified; no inference of its duration is authorized.

If the human is busy/unanswered, play a polite message that colleagues are occupied and someone
will call back as soon as possible, then end the call. Do not resume normal AI conversation.
The message itself does not create an automatic callback; follow-up visibility/delivery must be
specified before claiming a callback workflow. Localized wording follows the configured agent language.

The user prefers a simple billing approach and permits AI minutes if simpler. Inspection shows
current metering uses Vapi startedAt/endedAt and SUB-005 ends at provider end/transfer. Recommendation:
retain AI-pool metering for provider-reported AI-call duration, with no switch to operator minutes.
This does not establish billing for the remaining human conversation. Charging that duration to
either pool would still require authoritative carrier-leg correlation and reconciliation; it is
not made simple merely by choosing the AI pool. No new commercial policy is inferred here.

Technical discovery: Vapi documents that Twilio blind/non-assistant warm transfers relinquish
the AI and cannot play an AI fallback after failed handoff. Inbound's BYO SIP path and outbound's
Twilio path therefore need transport-specific transfer handling. Assistant-based warm transfer
supports fallbackPlan.message with endCallEnabled=true, but is documented as experimental and
must be tested on actual transports. The earlier direct-transfer recommendation is superseded
where it cannot satisfy the accepted failed-transfer message.

Sources: this conversation; current VapiCall and CallUsageService source;
[Vapi transfer failure behavior](https://docs.vapi.ai/phone-calling/in-call-control/transfer-calls/debug-forwarding-drops),
[assistant-based warm transfer](https://docs.vapi.ai/calls/assistant-based-warm-transfer), SUB-005.

### AGENT-003 Shared agent catalogue, configuration and Vapi warm transfer

**Subsequent transfer change, 2026-11-25:** [AGENT-008](#agent-008-shared-localized-spoken-handover)
replaces this record's native inbound transfer behavior and both routes' acceptance wording in the
checkout. Common agent fields/catalogues and mandatory localized fallback remain in effect.

**Source/approval date:** 2026-11-22. **Origin:** Explicit user implementation authorization and choice
of the first (Vapi) transfer approach, with live testing after implementation.
**Status:** Implemented in the API checkout; final verification recorded below; not deployed.

Use separate role-protected catalogue endpoints under `/api/agents`: languages, industries, purposes,
information-fields, time-zones, voices and voices/preview. Remove former voice roots and mixed inbound
options; expose authorized inbound customer choices separately. Do not expose templates as an endpoint.
Inbound and outbound have their existing separate campaign mutation endpoints but share an
AgentConfiguration value and prompt/tool construction. Industry/purpose/language templates are backend-owned.

Configuration includes required language, industry, purpose, first message, instructions, handover conditions
and international transfer number. Information collection is optional and appends descriptions only when
supplied. Inbound requires explicit voice selection. Retain the existing optional outbound platform-default
voice contract; do not adopt the original proposal to remove that default. Synchronize authorized outbound
language/voice changes into saved configuration and reject conflicting duplicate inputs. Inbound/outbound
schedules and operator/Collections behavior retain their existing representations and semantics.

Technical choices: frozen version-1 foundation/purpose resources, industry guidance fragments, and
configuration version references; preserve older versions for explicit upgrades. Inbound desired/applied/
replacement settings and outbound existing JSON storage carry the shared value, without new columns.
Outbound reservations retain their existing full request snapshots. No automatic data conversion/reset is
performed; old configured agents require explicit reconfiguration. The user confirmed local inbound storage
is empty; remote data remains uninspected. Drain old writers/workers and coordinate client cutover.

Both builders attach an explicit warm-transfer-experimental transferAssistant and a fixed validated human
destination. Human acceptance connects; voicemail/IVR/decline cancels. Busy/failure uses the localized
colleagues-busy/callback message and ends. The transfer assistant uses resolved speech settings and a focused
model prompt with only its provider/model configuration; unrelated tools/provider extras do not leak into it.
An explicit supported pipeline model is required; GPT-Live is excluded. Callback messaging does not create
an automatic callback job. Inbound history recognizes documented warm-transfer failures and does not infer
successful connection from forwarding. Existing subscription accounting is unchanged: no operator-minute
switch, double-counting or invented final human-leg duration is introduced.

No live provider mutation, transfer pilot, callback automation, frontend source change, commit or deployment
is included. Actual Twilio/SIP behavior, speech quality, post-transfer recording/duration and billing/control
boundaries remain explicit live-test gates. The user accepted experimental transfer for this implementation;
local payload tests do not establish its production behavior.

Sources: this conversation; [API contract and live-test checklist](../../apps/api/docs/api/shared-agent-configuration-web-handoff.md);
[implementation plan](../../apps/api/docs/superpowers/plans/2026-11-22-shared-agent-configuration.md);
[Vapi guide](https://docs.vapi.ai/calls/assistant-based-warm-transfer);
[official transfer-assistant type](https://github.com/VapiAI/server-sdk-typescript/blob/main/src/api/types/TransferAssistant.ts).

Final local verification, 2026-11-22: MySQL-enabled `./gradlew test bootJar` passed with 927 tests:
926 passed, zero failures/errors, one opt-in live Vapi catalog test skipped. Coverage includes all
88 advertised language/industry/purpose combinations in both directions, catalogue roles/routes,
warm-transfer payload/speech/fallback/model constraints, transfer outcomes, JSON persistence,
configuration/voice synchronization and rejection of activation with missing mandatory handover.
One final self-review found and fixed the inbound activation guard; this is not an independent-agent
review or live-call verification. No deployment, provider mutation or UI verification was performed.

### AGENT-004 Consolidate catalogue HTTP handlers

**Decision/source date:** 2026-11-22. **Origin:** Explicit user request to move voice endpoints into
AgentCatalogController rather than retain a separate controller.
**Status:** Implemented in the checkout; focused contract/OpenAPI checks recorded below. Not deployed.

AgentCatalogController owns metadata, voice listing and previews. Remove VoiceCatalogController;
retain VoiceCatalogService and provider/media services as separate business/integration responsibilities.
URLs, operation IDs, query/response contracts, preview headers and role checks remain unchanged.
Swagger groups the endpoints under Agent catalog. This is controller organization, not another catalogue
implementation, provider call or client contract migration.

Sources: this conversation; [current handoff](../../apps/api/docs/api/shared-agent-configuration-web-handoff.md).

Verification: `./gradlew test --tests '*VoiceCatalogApiTest' --tests '*CampaignServiceCreateCampaignTest' bootJar`
passed 44 tests with zero failures/errors/skips, including voice/catalogue roles, preview headers/errors,
removed old routes and generated OpenAPI contracts/operation IDs. The full MySQL suite was not rerun
for this controller-only follow-up; AGENT-003's 927-test run remains historical evidence for its own revision.

### Check-in verification for AGENT-003/004

**Verification date:** 2026-11-22. **Scope:** Final combined API checkout, not deployment.

The check-in review corrected a stored configuration with null industry to return
`invalidAgentConfiguration` rather than a null-pointer failure, with a regression test. Current
inbound examples now use nested configuration and the shared voice routes. The shared handoff
also documents existing outbound number selection and assignment; no inventory API contract changed.

Full MySQL-enabled `./gradlew build` passed after the fix and controller consolidation:
928 tests total, 927 passed, zero failures/errors, one opt-in `VapiVoiceCatalogLiveTest` skipped.
Both plain and Spring Boot JARs built. The web checkout was clean and was not rebuilt/tested.
Real Twilio/SIP transfers, human-leg billing, frontend integration and deployment remain unverified.
No commit or staging change was performed. This updates the verification evidence for AGENT-003/004;
their preceding run results remain historical.

Source: the user's check-in readiness request in this conversation; [current API handoff](../../apps/api/docs/api/shared-agent-configuration-web-handoff.md).

### Subsequent local delivery

2026-11-22: the user explicitly requested local commits for the API and then the parent documentation
and submodule pointer, without pushing. The API implementation and handoffs were committed as
`3e4c0ee` (`feat(api): unify agent configuration and Vapi handover`) on
`codex/shared-agent-configuration`, based on the tested `310805e` checkout. The parent gitlink is
updated to that API commit in this delivery. The separate API `master` logging commit is not part
of this tested change. The verification results above apply; no push, deployment or live call occurred.

### Integration with API master

**Source/verification date:** 2026-11-23. **Origin:** Explicit user request to merge `master` into
`codex/shared-agent-configuration`. This is integration of existing decisions, not a new API contract.

The earlier delivery above was subsequently pushed. API merge commit `894aec5` now combines
`3e4c0ee` with `master`'s `ff42830` logging changes. The overlapping create/update sections in
CampaignMutationService retain both shared configuration normalization/validation and the logging
stage markers. Shared catalogue, template, voice and transfer behavior remain in place. The other
overlapping changes merged automatically and were reviewed. `master` is an ancestor of the feature
branch after this merge; the feature branch has not been merged into `master`.

Full MySQL-enabled build of the combined checkout passed: 940 tests total, 939 passed, zero
failures/errors, one optional `VapiVoiceCatalogLiveTest` skipped. The initial default-heap run
exhausted Java heap; verification succeeded using a temporary Gradle init script setting the test
task's maxHeapSize to `2g`. No repository build configuration was changed. Both JARs built; real
Vapi calls, transfer billing and deployment remain unverified.

The parent gitlink is updated locally to `894aec5` with this documentation. No push or deployment
is included in this merge follow-up. Sources: this conversation and the merge/build results above.

### Inbound transfer investigation — 2026-11-24

**Origin:** User report of two failed inbound transfers after successful terminal tests.
**Status:** Read-only production investigation; inbound transfer remains an unresolved live-test gate.
No routing fix, provider mutation, test call, commit or deployment was performed in this investigation.

Retrieved Vapi call records, authenticated call logs and SIP captures for
`01a20d2a-0d52-7ee4-a440-5f1143859cc6` (10:46 UTC) and
`01a20d5e-8f28-7000-957d-0339d522bcbe` (11:43 UTC). Both invoked transferCall to the
configured destination, returned “Transfer initiated”, entered warm-transfer PENDING and ended
about 0.1 seconds later with `customer-ended-call-after-warm-transfer-attempt`. Neither record
has an operator conversation or completed transfer. The captured SIP dialog shows Vapi-side BYE
towards Twilio, with no preceding caller-side BYE or REFER in that capture. The endedReason alone
therefore does not establish that the human caller hung up. Absence of REFER does not establish
a provider rejection; Vapi-managed numbers may dial rather than send REFER.

These calls use NexCall's Twilio Dial/Sip bridge into a Vapi-managed SIP URI (provider vapi,
transport sip; no PSTN number or linked trunk credential on that Phone Number resource).
The current assistant includes the configured warm transfer assistant/fallback; serverMessages
does not delegate phone-call-control to NexCall. Twilio read-only records show only the original
inbound call and SIP child, no operator child call and no notification errors. NexCall production
logs show final usage settlement; they do not explain the provider-side termination.

By comparison, terminal test `01a20956-b28e-7000-b487-9a50d0a16590` on 2026-11-23 used
an imported Twilio PSTN number and reached warm-transfer ACTIVE, operator acceptance through
transferSuccessful, and an explicit connected transfer record. That verifies that test's transfer
connection only, not the inbound SIP route or subsequent human-leg metering.

Conclusion: transfer configuration is present, but successful direct-Twilio tests do not validate
the deployed inbound SIP-to-PSTN transfer path. Exact Vapi-side termination/outbound-leg cause is
not exposed in retrieved logs. Next verification must use the actual inbound route and inspect
Vapi's destination-leg/internal trace; any routing correction must preserve tenant admission,
call correlation, failed-transfer behavior and usage controls. Changing sipVerb alone is not a
verified fix, and adding a Twilio REFER handler cannot explain the missing REFER in this capture.
No new product or billing policy is inferred.

Sources: the above live Vapi artifacts, matching Twilio call/notification reads, current
TwilioService inbound TwiML and AgentAssistantConfiguration, and Vapi's
[transfer troubleshooting](https://docs.vapi.ai/phone-calling/in-call-control/transfer-calls/debug-forwarding-drops),
[warm-transfer guide](https://docs.vapi.ai/calls/assistant-based-warm-transfer) and
[authenticated artifact retrieval](https://docs.vapi.ai/assistants/retrieve-call-artifacts).

### AGENT-005 Preserve NexCall SIP entry for inbound transfer

**Source/decision date:** 2026-11-24. **Origin:** Explicit user requirement to retain
`Caller -> Twilio -> NexCall -> Vapi SIP -> agent` and find a transfer correction within it.
**Status:** Routing constraint accepted; investigation documented. A specific transfer correction
has not been selected, implemented or live-verified. Native Twilio inbound routing that bypasses
NexCall's current entry path was proposed earlier in chat and is not the selected direction.

Preserve NexCall's incoming Twilio webhook, admission/tenant checks, campaign schedule, SIP handoff
and call correlation. Existing human acceptance and localized failed-transfer behavior remain
requirements; a blind transfer is not an implicitly approved substitute.

Read-only provider inventory found no Vapi BYO SIP trunk credentials and no Twilio Elastic SIP
trunks. The current managed SIP target has no linked outbound trunk. A candidate is to configure
an authenticated outbound SIP route for Vapi's operator leg and validate assistant-based warm
transfer while retaining NexCall's inbound bridge. The source of the immediate hang-up is still
unconfirmed; missing trunk configuration alone does not prove causality. Compatibility of the
combined ingress/egress arrangement must be checked before claiming a fix.

An alternative is NexCall/Twilio-controlled transfer, with more orchestration to preserve warm
operator acceptance and fallback. SIP REFER would need a referUrl handler; adding one alone is
insufficient evidence because no REFER was present in the failed-call captures. No provider
resources, production settings or application code were changed for this investigation.

The proposed [SIP transfer validation plan](../../apps/api/docs/superpowers/plans/2026-11-24-inbound-sip-transfer-validation.md)
records candidates, acceptance criteria and primary references. Outbound calling remains unchanged.

### AGENT-006 Evaluate server-controlled inbound handover

**Source/decision date:** 2026-11-25. **Origin:** User proposal: Vapi notifies NexCall when a
human transfer is requested, and NexCall uses Twilio to add/bridge the operator while retaining
the existing inbound route. **Status:** Feasible architectural proposal under discussion;
no runtime implementation, provider change or live verification. This is not approval to replace
the accepted conversational warm transfer with a blind or keypad-only flow.

Technical recommendation: evaluate a targeted inbound custom Function tool that sends an
authenticated HTTPS tool-calls command to NexCall. A second documented option is phone-call-control,
which delegates forwarding and hang-up together and therefore requires both handlers. SIP media
does not automatically deliver an application transfer event, and a browser socket is not required.
The current status/end-report-only webhook does not implement either transfer command path.

NexCall would correlate the original caller leg, hold it, dial the trusted configured operator
through Twilio, confirm acceptance privately and bridge the parties. Existing inbound calls use
Dial/Sip, not a conference; a safe transition and durable/idempotent transfer lifecycle are needed
before conference participant APIs can be used. Preserve tenant boundaries, caller continuity,
failed-transfer message/hang-up, call limits, history and billing boundaries. Private AI operator
introduction/acceptance remains an explicit design item, not something a conference supplies by itself.

The updated [validation/design plan](../../apps/api/docs/superpowers/plans/2026-11-24-inbound-sip-transfer-validation.md#server-controlled-transfer-proposal--2026-11-25)
now recommends evaluating this candidate before adding an outbound SIP trunk solely for transfer.
The earlier trunk recommendation was a test hypothesis, not an established best fix. Ordinary
Twilio Programmable Voice dialing/conferencing does not require an Elastic SIP trunk.
No commercial policy or production rollout is authorized by this feasibility discussion.

### AGENT-007 Prototype NexCall-controlled inbound transfer with keypad acceptance

**Subsequent scope change, 2026-11-25:** [AGENT-008](#agent-008-shared-localized-spoken-handover)
supersedes the allowlist, keypad-only acceptance and 60-second human cap in the checkout. The
following pilot deployment and live results remain historical evidence; the new flow is not yet deployed.

**Source/decision date:** 2026-11-25. **Origin:** Explicit user agreement to test
agent -> Vapi transfer request -> NexCall call/destination lookup -> Twilio operator dial/bridge;
the user explicitly answered **Allow press 1 for this prototype**. **Status:** Implemented, verified
and deployed for a campaign-70-only pilot with one successful live keypad-acceptance transfer and
user-confirmed two-way audio; source delivery and pilot evidence are recorded below.

This temporary test exception permits keypad acceptance after a private introduction. It does not
replace the accepted production conversational warm transfer or change outbound behavior. Keep
AGENT-005's incoming route and original NexCall admission checks. The prior SIP-trunk candidate
is not a prerequisite or the selected implementation of this prototype.

Technical choices: an authenticated inbound-only Function tool, campaign allowlist and default-off
feature switch; authoritative Vapi/Twilio identity and admitted-route verification; destination from
applied agent settings; commit durable operation intent before the provider command; unique caller
and Vapi identities; no automatic replay of uncertain redirects. Use Twilio Dial/Number screening
with Gather/press-1 rather than introducing a conference. Signed callbacks establish acceptance
and final bridge outcome; retain localized fallback plus hang-up for rejection/unreachable operator.

The test caps the human bridge at 60 seconds and conservatively within the original call deadline.
Vapi history/recording and existing AI metering end with the AI leg at Twilio takeover. This prototype
does not add human-leg recording, invoice/minute policy or UI history integration, and has no new
reconciler for missing final transfer callbacks. These are explicit production follow-up items.
The operational [activation and validation plan](../../apps/api/docs/superpowers/plans/2026-11-24-inbound-sip-transfer-validation.md#opt-in-prototype-and-live-validation--2026-11-25)
documents migration 040, configuration, reprovisioning, state meanings, rollback and live-test gates.

Verification, 2026-11-25: full MySQL-enabled API tests and bootJar passed: 1,237 total tests,
1,236 passed, no failures/errors, 250 MySQL tests and one opt-in live voice-preview test skipped.
After the final exact-fallback prompt clarification, 56 focused tests passed (18 MySQL), no skips,
and bootJar passed again. XML and whitespace checks passed. The first broad run identified a
missing optional RestClient.Builder bean; the corrected client uses it when available and otherwise
matches the application's existing manual-builder setup. Real caller continuity, DTMF acceptance,
operator audio, provider failure behavior and rollback still require the controlled live pilot.

Pilot selection, 2026-11-25: the user explicitly chose campaign **70, AgentulX** using the
Kubernetes connection. Read-only checks confirmed READY provisioning, IDLE campaign status, active
tenant/subscription, Romanian language, saved inbound number ending 0630 and transfer destination
ending 7177. The deployed API remains 023e7a5 without migration 040 or prototype settings. Use only
campaign ID 70 for opt-in; preserve its existing numbers and record IDLE as the rollback status.
No campaign activation, provider resource mutation, source delivery or deployment occurred during
preflight. The validated local JAR is built; a new API build must be deployed before a live pilot.

### Prototype source delivery and deployment authorization — 2026-11-25

The user explicitly requested pushing the changes and performing the live test rather than a
local-tunnel pilot. API `master` commit `2a48021` (`feat(inbound): add opt-in server-controlled
transfer prototype`) is pushed. The parent `main` delivery includes the investigation/decision
records and this API gitlink. The unrelated web checkout is excluded.

The API push triggers its existing Deploy GKE Prod workflow. CI/deployment results and actual
audio/transfer outcomes are not inferred from the source push. After the new build and migration
040 are verified healthy, scope the prototype to campaign 70, reprovision AgentulX through the
normal update flow, and activate it for the controlled pilot. The original IDLE state is recorded
for rollback. Acceptance for this pilot remains the user's explicit press-1 choice.

### Prototype deployment and pilot readiness — 2026-11-25

API master `2a48021` is deployed. GitHub workflow
[45597288392](https://github.com/red-abacus/nexcall-api/actions/runs/45597288392) completed successfully:
tests, chart lint/render, image build/push, Helm deployment and post-deploy smoke checks passed.
Liquibase applied migration 040 successfully, and the prototype endpoint accepted an authenticated
preflight while rejecting the deliberately malformed command with HTTP 400 before any call request.

After the CI rollout, a scoped Kubernetes environment update enabled the prototype with
`NEXCALL_INBOUND_TRANSFER_PROTOTYPE_CAMPAIGN_IDS=70`. Its follow-up rollout completed successfully;
the only serving API pod runs the new image and the allowlist is 70 only. These pilot environment
overrides are temporary; the current Helm values do not persist them across future deployments.

AgentulX's assistant was backed up privately outside the repository. For this controlled pilot,
only its model handover tool/prompt and server message list were patched through the Vapi API to
the same requestHumanTransfer profile built by the new provisioning code. Readback verified the
authenticated NexCall tool URL, no native transferCall tool, and unchanged voice, transcriber and
first message. No Twilio incoming route, SIP endpoint or saved agent destination was changed.
This direct provider patch avoided bypassing API authentication or editing lifecycle rows in SQL.

At handoff the campaign remains IDLE. Browser control permissions were unavailable, so the user
was asked to activate AgentulX normally in the UI and place the inbound test call, then accept on
the configured operator phone with 1. No live transfer outcome is claimed until a real request,
provider leg and observed caller/operator result are available. The original IDLE state and
assistant snapshot remain the rollback reference.

### Live prototype result — 2026-11-25

The user activated AgentulX (70) through the UI and performed two real incoming calls through
the retained Twilio -> NexCall -> Vapi SIP route. No Elastic SIP trunk or incoming-route change
was required for this prototype.

- First attempt: Vapi call `01a21258-d37b-7000-b480-48d9aee28efe`, transfer operation
  `11760209-9284-446e-b6f6-61bb1059f457`. The tool returned HTTP 200, Twilio created/answered
  operator leg `CA682197dab3401e0743d6615c6a14aa6d`, and screening returned HTTP 200. No acceptance
  callback was received; the final action recorded FAILED/connected=false and served fallback
  TwiML. The original caller leg `CAb07feeb16d47b1aea72073680758c4ed` lasted 39 seconds,
  continuing after Vapi's AI leg ended. No Twilio notification errors were recorded. This does
  not establish why no digit was received or independently confirm that the caller heard fallback.
- Second attempt: Vapi call `01a21259-92d6-7000-ad2d-938765777d59`, transfer operation
  `df942677-c483-42f0-976d-a93da55d8464`. The workflow progressed DIALING -> SCREENING -> ACCEPTED
  -> CONNECTED. Screening, acceptance and final-action callbacks returned HTTP 200. The stored
  CONNECTED state requires recorded acceptance, the matching operator SID and DialBridged=true.
  Caller leg `CA22eb92a63273cddc8a39f260892dbad3` completed after 44 seconds; operator leg
  `CAe9e52435f48274d16747c13b39d231db` completed after 20 seconds, with no Twilio notification
  errors. These durations include their respective setup/screening and are not a new billing rule.

The user explicitly reported that the transfer worked, confirmed audio in both directions and
accepted the approach as a good one. This verifies the server-controlled, press-1 prototype on
campaign 70. It does not verify the former native Vapi SIP warm transfer, spoken AI acceptance,
all failure cases, human-leg recording/billing or a general production rollout.

Current pilot state: the user-activated campaign remains ACTIVE, only campaign 70 is allowlisted,
and the prototype's 60-second human-bridge limit remains configured. No other campaign's transfer
profile was changed. The original IDLE state and private assistant snapshot remain available for
rollback; rollback has not been applied. Further production scope requires a separate decision.


### AGENT-008 Shared localized spoken handover

**Source/decision date:** 2026-11-25, Europe/Bucharest. **Origin:** Explicit user requirements
following the successful pilot: enable all inbound agents, remove the 60-second human-call cap,
share caller/failure messages across directions, and instruct spoken YES/NO or DA/NU according
to campaign language. **Status:** Implemented and locally verified; API source pushed to master as `98730bc`.
Deployment, Romanian inbound acceptance/no-answer/refusal and Romanian outbound acceptance/audio
are verified; English, outbound negative cases and extended human duration remain pending. Supersedes AGENT-007's pilot scope, keypad acceptance and human cap;
its historical proof and unresolved billing/recording limitations remain valid.

The common message catalogue now drives both routes. Inbound keeps AGENT-005's NexCall SIP entry
and uses Twilio private speech Gather before bridging. Outbound retains its working Vapi native
experimental transfer assistant with explicit affirmative-word acceptance/cancellation instructions.
The operator introduction explains acceptance, refusal and connection after YES/DA. Failed/no-answer
transfers deliver the same localized colleagues-busy/callback message and end the caller's call.
No automatic callback job is introduced. Campaign language selects all three messages.

Technical choices: replace the default-off allowlist with a global default-on inbound handover
switch; preserve resource/admission checks and operation idempotency. Only an entire affirmative
word in the campaign language permits inbound bridging, with case/punctuation normalization;
unknown, negative, empty and keypad responses cannot accept. Outbound applies the same rule in
its Vapi assistant prompt, so model adherence and recognition remain provider-dependent. Twilio
Romanian speech recognition is documented but this new spoken flow is not established by the
previous live keypad pilot. Voice timbre/transport and internal timing are not identical across providers.

The human leg can exceed 60 seconds while respecting the original campaign/provider duration
budget and conservative setup reserve. Existing AI usage ends with its provider leg; no new human-leg
billing, recording or operator-minute policy is inferred. Missing callback reconciliation remains open.
There is no new schema or client request field. Existing Vapi inbound assistants require normal
reprovisioning after deployment; all future outbound call snapshots use the new shared configuration.
Already-running calls are unchanged. Provider refresh and live testing are tracked separately from implementation.

Sources: user request in this conversation; API [rollout and validation plan](../../apps/api/docs/superpowers/plans/2026-11-24-inbound-sip-transfer-validation.md#shared-spoken-handover--2026-11-25);
[client handoff](../../apps/api/docs/api/shared-agent-configuration-web-handoff.md#prompt-and-handover-behavior);
[Twilio speech Gather](https://www.twilio.com/docs/voice/twiml/gather);
[Vapi transfer configuration](https://docs.vapi.ai/calls/assistant-based-warm-transfer).


Verification, 2026-11-25: MySQL-enabled `./gradlew build` passed with 1,248 total tests,
1,247 passed, zero failures/errors, 254 MySQL tests and the optional `VapiVoiceCatalogLiveTest`
skipped. After the final greeting/speech-capture adjustment, the focused transfer, assistant composer,
inbound provisioning and outbound request-factory selection passed all 155 tests (22 MySQL), no
skips, and `bootJar` passed again. The temporary 2 GB test-heap init script was used; no project
heap/toolchain/dependency change was introduced. Whitespace checks and new documentation link
targets passed. These are local payload, HTTP security and persistence checks, not live RO/EN
recognition, Vapi model compliance or carrier-audio verification. Deployment/provider refresh and
the controlled spoken-transfer matrix remain outstanding release steps.


### Shared spoken handover delivery — 2026-11-25

The user explicitly requested committing/pushing and live tests in both directions. API master
`98730bc181a8228c1944560897f0a507d6f30574` is pushed; its normal production workflow
[45605127236](https://github.com/red-abacus/nexcall-api/actions/runs/45605127236) is in progress.
Parent main includes the decision/handoff references and API pointer; unrelated web changes are excluded.
Do not infer deployment or successful spoken acceptance from the source push. Verify the deployed
image/readiness, drain old keypad calls, refresh inbound profiles and test both real call paths.


### Shared spoken handover deployment and test readiness — 2026-11-25

Workflow [45605127236](https://github.com/red-abacus/nexcall-api/actions/runs/45605127236)
completed successfully, including CI tests, Helm lint/render, image build/push, deployment and
post-deployment smoke checks. The serving image is API `98730bc181a8228c1944560897f0a507d6f30574`;
readiness is UP. The obsolete prototype-enabled/campaign-70 environment overrides persisted through
Helm's merge and were explicitly removed afterward. That cleanup rollout completed with one ready,
updated replica. The new default-on transfer configuration has no campaign allowlist.

The user selected outbound campaign **71, Test**, in addition to inbound **70, AgentulX**. Read-only
preflight found no active pilot calls. AgentulX remained ACTIVE/READY in Romanian, but provider
readback still showed its old pilot profile after the user reported READY. To prepare the selected
live test, its assistant was backed up privately outside the repository and only model messages,
model tools and serverMessages were patched through the authorized Vapi API. The payload was built
using the committed AgentAssistantConfiguration/InboundTransferProperties code from the verified
local JAR and the agent's applied configuration. Readback verified the new prompt and silent tool
start; voice, transcriber, first message, artifact plan, call duration and server configuration were
unchanged. No lifecycle SQL writes or API authentication bypass were used. This is a provider refresh
for campaign 70, not evidence that other existing assistants have been reprovisioned.

Both selected campaigns currently use Romanian. Outbound 71 has one idle task (76), recipient ending
7177, transfer destination ending 9510. Inbound uses its saved destination ending 7177. The controlled
inbound test is requested first: caller on a different phone, private DA acceptance, two-way audio and
at least 75 seconds of human conversation after connection. Outbound 71 remains idle until the
inbound call finishes. No spoken-transfer outcome or English live verification is claimed yet.


### Romanian inbound spoken-transfer results — 2026-11-25

Three real AgentulX (70) calls exercised the deployed NexCall -> Twilio bridge with the shared
Romanian messages. The user supplied audio/experience confirmation; provider/application observations
are distinguished below. Outbound campaign 71 remained idle during these inbound checks.

- Acceptance: Vapi `01a2128a-c5b3-7000-bba4-e74b5b797cf6`, operation
  `f0c6d0b8-6dc9-4ca3-9bb4-f430d3aaabab`, caller `CA3a99600840210dd4b604c32b89afbaea`,
  operator `CA8751f5f951fd9ca98c38ffdf87e85642`. Acceptance was stored and the final callback
  recorded CONNECTED (matching operator and DialBridged=true). The user confirmed the Romanian
  introduction, spoken DA acceptance and audio worked. Only approximately 9 seconds elapsed between
  accepted_at and finished_at: this does not verify human conversation beyond 60 seconds.
- Unanswered operator: Vapi `01a2128c-cd3b-7000-92f2-07a1e9d70b3a`, operation
  `846b93c0-7e15-4990-92a5-2bab249c4ae6`, caller `CAb056ff67cca972909631dbf33842c342`.
  No acceptance was stored; final state was FAILED/connected=false. The user explicitly confirmed
  the expected Romanian colleagues-busy/callback message followed by call termination.
- Spoken refusal: Vapi `01a2128f-464c-7000-b853-8f3c5507cac5`, operation
  `b6fbc0e3-975f-44eb-9d7c-0a881a380536`, caller `CA2f98418ee14bcc0ce32c043b2ca83e4c`,
  operator `CA5e3c1cb38a17d522632b41cdcea99d8f`. Twilio events showed HTTP 200 screening,
  acceptance and completion callbacks; the acceptance request's speech_result was exactly `nu`.
  The operator response contained Hangup, and the caller completion response contained Romanian
  fallback plus Hangup. No acceptance was stored and final state was FAILED/connected=false.
  The user confirmed no operator/caller connection, correct fallback audio and termination.

Subsequent outbound preflight found a second live idle contact in campaign 71: task 77 ending
9510, in addition to task 76 ending 7177. Its saved transfer destination still ends 9510. The user
was asked to select the intended caller/operator pair before activation, to avoid treating an
unintended second campaign call or a transfer to the current recipient as the controlled test.
No contact, task or campaign was deleted or modified by the agent. Outbound and English live
verification, >60-second human audio, voicemail/ambiguous-input cases and remaining production
follow-ups are not implied by the three successful Romanian inbound checks.


### Romanian outbound spoken-transfer result — 2026-11-25

Before activation, the user explicitly reported updating the direct/transfer phone number in
campaign 71. Read-only checks verified its saved transfer destination now ends **7350**, distinct
from both existing recipients (task 76 ending 7177 and task 77 ending 9510). The campaign remained
Romanian and was activated through the normal UI flow. The agent made no task/campaign SQL writes.

NexCall's deployed worker reserved and submitted two actual outbound attempts to Vapi:

- Attempt **226**, task 76, Vapi `01a21294-b114-788e-858f-52e4a4c4cd84`: ENDED with
  `customer-ended-call`. Provider messages contained endCall, not transferCall. This first call is
  not counted as a transfer validation. Its task reached DONE normally.
- Attempt **227**, task 77, Vapi `01a21295-28dd-7000-9639-1d5daecc465f`: the immutable request
  snapshot contained the shared Romanian caller message, complete DA/NU operator introduction,
  colleagues-busy/callback fallback, Romanian transfer-assistant transcriber and destination ending
  7350. The worker logged provider acceptance, Vapi messages contained transferCall, and the call
  ended with `assistant-forwarded-call`; its task reached DONE. Forwarding alone is not proof of
  operator connection. In response to the explicit live-test question covering the caller message,
  operator instructions, spoken DA and audio both ways, the user confirmed **worked as expected**.
  This user observation supplies the audio/acceptance evidence beyond the forwarding status.

The normal NexCall outbound dispatch, rather than a hand-built terminal call, therefore exercised
the new shared configuration. Romanian successful handover is live-verified for both incoming SIP/
Twilio server-controlled routing and native outbound Vapi warm transfer. Inbound no-answer and NU
refusal are additionally verified above. English, outbound no-answer/refusal, >60-second human
conversation, ambiguous speech/voicemail and existing follow-ups remain unverified or unresolved;
no full transfer matrix, new human-leg billing or recordings policy is inferred. Other saved inbound
assistants still require normal provisioning to adopt the new profile. The implementation-time
release gate in the API plan is historical; these canonical deployment/live records establish the
subsequent verification state.


### AGENT-009 Human transfer outcome on each inbound interaction

**Superseded draft:** [AGENT-010](#agent-010-common-human-transfer-history-contract) replaces the
unpublished inbound-only raw-state/id/transferred response with the common normalized contract.
The original draft's verification below is retained as historical evidence, not the final contract.

**Source/decision date:** 2026-11-25, Europe/Bucharest. **Origin:** Explicit user requirement to
extend the interactions response so clients can display human transfer information for each call.
**Status:** Implemented and locally verified in the API checkout. Not committed,
pushed, deployed or integrated into web/mobile. This implements the interaction-read portion of
the formerly deferred transfer-history integration; actual client rendering remains separate.

`GET /api/inbound/agents/{id}/interactions` adds nullable `humanTransfer` to every item. A matching
operation supplies its ID, raw status, saved transferNumber, requestedAt, nullable acceptedAt and
finishedAt, and a transferred boolean. Only persisted CONNECTED (acceptance, matching operator,
DialBridged=true in the completion callback) sets transferred=true. Pending acceptance, DECLINED,
FAILED and UNCERTAIN do not establish a successful bridge. A null object means no matching
NexCall-managed operation; historical provider transferStatus remains meaningful when present.
This is an additive response change: existing interaction/provider status fields and IDs are retained.

Technical choice: enrich the bounded interactions page in one SQL query using the existing
inbound_human_transfer table. Correlate within the campaign using the Vapi call ID and/or original
Twilio caller SID; when both identities are present, both must match. Do not join by campaign alone,
match contradictory identities, expose internal provider SIDs or fetch a provider per interaction.
No new entity, migration, transfer callback write or persisted JSON copy is needed. Reads expose
updates and previously retained transfers without a backfill. Existing campaign authorization and
cursor pagination remain in force. The destination reflects the request-time snapshot rather than
subsequent agent edits. Interaction timestamps retain UTC ISO strings.

Clients must use the status as well as transferred: false during ACCEPTED/UNCERTAIN does not
prove failure. Final bridge confirmation generally arrives when the operator leg finishes, so an
AI interaction can be completed while its transfer is pending. Missing-final-callback reconciliation,
more detailed failure reasons, human-leg recordings/billing and a new interaction notification event
are not introduced. UI integration steps and exact state/null semantics are in the updated
[client handoff](../../apps/api/docs/api/inbound-agent-web-handoff.md#human-transfer-on-each-interaction).
The [source plan](../../apps/api/docs/superpowers/plans/2026-11-24-inbound-sip-transfer-validation.md#interaction-level-human-transfer-visibility--2026-11-25)
tracks this local extension separately from the deployed AGENT-008 transfer behavior.


Verification, 2026-11-25: the focused interactions, handover, callback-auth/logging and CRM integration
selection passed all 113 tests, including 69 MySQL tests, with zero failures/errors/skips; bootJar
passed. Tests cover all operation states, timestamp/null semantics, acceptance before confirmation,
callback updates without interaction-JSON writes, Vapi-only/Twilio-only matches, absent/contradictory
identities, another campaign/call, authorization before database access, stable IDs and cursor pages.
The initial run exposed mistakes in new fixture parameter binding and schema-name length; those
fixtures were corrected before the successful rerun. A read-only production query using the same
join matched existing AgentulX interactions 18–22 to their saved transfer operations, including
CONNECTED for interaction 20 and FAILED for 21/22. This verifies existing-data correlation, not
new endpoint deployment. No production writes, provider changes or new live calls were performed.
Whitespace and new local documentation link targets were checked. The full API suite was not rerun
for this read-only extension; the executed selection includes the real MySQL/CRM history flow.


### AGENT-010 Common human transfer history contract

**Source/decision date:** 2026-11-25, Europe/Bucharest. **Origin:** Explicit user requirement and
implementation authorization for the same client contract in both directions, despite different
collectors. The user accepted a normalized status and destination with available nullable timing,
rather than provider/internal operation details. **Status:** Implemented and locally verified;
API master `394cadd` is pushed. Deployment is queued, and web/mobile rendering remains separate.
Supersedes AGENT-009's unpublished response shape; deployed AGENT-008 calling behavior remains
the verified runtime baseline until the new rollout is confirmed.

Both inbound interaction items and outbound task call-history items expose nullable humanTransfer
with exactly status, transferNumber, requestedAt, acceptedAt and finishedAt. A shared DTO/enum
owns the wire shape. The user explicitly requested removing redundant JsonInclude(ALWAYS)
overrides: retain the existing Jackson null-inclusion default and verify it through contract tests,
without changing global serialization configuration. Statuses are in-progress, connected, no-answer, busy, voicemail, cancelled,
failed, completed and unknown. Only connected proves bridging. Completed is retained from Vapi
without upgrading it to connected. Operator acceptance is still in-progress until bridge confirmation.
The redundant transferred boolean and raw operation ID/states from the prior draft are removed.
Null means matching transfer information is unavailable; it is not proof that no transfer occurred.

Update, 2026-11-25: [SUB-024](2026-11-18-subscriptions-and-call-limits.md#sub-024-split-transferred-calls-between-ai-and-operator-allowances) establishes split billing and permits a verified outbound screening timestamp in `acceptedAt`. The original contract below records the earlier evidence.

Timestamps stay nullable UTC ISO values. Inbound uses its recorded operation timestamps; outbound
artifacts currently provide no separate requested/accepted/finished timestamps, so those values
remain null. Do not substitute whole-call start/end or derive a billable human duration. The number
is the request-time/provider destination, not a subsequently edited campaign setting. No provider
SIDs, transcript or mode is part of the shared client contract. Other call fields, identities,
permissions, collection wrappers, pagination, recordings and CRM actions remain in place.

Technical implementation: retain inbound campaign/call identity matching and enrich the bounded
history page on read. Migration 041 adds an outcome column so signed Twilio DialCallStatus callbacks
can retain no-answer/busy/cancelled, and the explicit NO/NU decision survives final FAILED operation
state. Ambiguous/missing spoken acceptance remains generic failed. Old rows lacking the reason
remain failed; voicemail is never inferred for inbound. Existing raw operation state remains usable
for control/audit. Completed callbacks remain idempotent.

For outbound, extend Vapi observations with normalized artifact/forwarding evidence and store a
minimal JSON summary on each VoiceAgentAttempt. Existing call/org/provider-number/recipient
validation precedes updates. A forwarded-call end reason alone yields unknown, not connected.
Artifact connected was observed on the actual campaign-71 test call and is distinct confirmation
from that end reason. Missing/late/weaker observations do not erase confirmed connection evidence.
Read destination fallback from the immutable request; malformed legacy metadata cannot prevent
processing the provider's call outcome. API history reads do not call providers.

A separate metadata check lane uses existing Vapi credentials and worker cadence. It handles
ended calls from the last 30 days, at most five per tick, with row-locked claims before provider
GETs and 20-second retry spacing for at most two minutes from first claim. Private checked/next
markers added in migration 041 prevent duplicate concurrent claims and keep this independent of
submission/disposition queues and older replicas. Known results, no transfer and missing provider
calls stop checks; exhausted unknown outcomes remain unknown. New authenticated observations can
still add stronger evidence later. This is a technical, bounded history-refresh choice, not a new
calling/billing policy or promise of reconstructing expired provider data.

Deliver one [web handoff](../../apps/api/docs/api/human-transfer-web-handoff.md), linked from the
inbound and shared-agent handoffs, containing only endpoints, response fields, examples, display,
refresh and integration verification rules. [Backend rollout notes](../../apps/api/docs/deployment-gke.md#human-transfer-history-contract)
and the [source plan](../../apps/api/docs/superpowers/plans/2026-11-24-inbound-sip-transfer-validation.md#interaction-level-human-transfer-visibility--2026-11-25)
keep migration, collection mechanics and runtime status separate from the client instructions.

Sources: explicit user agreement/implementation request in this conversation; existing inbound
operation/callback implementation; private read-only Vapi GET of the campaign-71 test artifact;
[Vapi TransferArtifact](https://github.com/VapiAI/server-sdk-typescript/blob/main/src/api/types/TransferArtifact.ts),
[Vapi status values](https://github.com/VapiAI/server-sdk-typescript/blob/main/src/api/types/TransferArtifactStatus.ts),
[Vapi phone transfer events](https://docs.vapi.ai/server-url/events#phone-and-sip-transfers).


Verification, 2026-11-25: MySQL-enabled `./gradlew build` passed with 1,333 total tests,
1,332 passed, zero failures/errors, 286 MySQL tests and one optional live voice-catalog test skipped.
After the user's requested removal of JsonInclude overrides, the focused transfer, worker, history,
callback-security and persistence selection passed all 236 tests (83 MySQL), zero failures/errors/
skips, and bootJar passed again. The temporary 2 GB test-heap init script was reused; no project
heap/toolchain/dependency change was made. Contract tests confirm exact five-field JSON and explicit
nulls using default serialization. Real MySQL tests confirm migration and JSON persistence, while
Spring history tests confirm per-attempt exposure, scope validation and stale-update protection.
New metadata checks are tested for claim timing, retry exhaustion, active/old/missing calls and
operation without dispatch; new inbound tests retain busy/no-answer/cancellation and distinguish
literal NO/NU from ambiguous/missing acceptance. XML parsing, whitespace and local link targets pass.
No new live calls, production migration, provider configuration changes or deployment were performed
for AGENT-010. End-to-end observation of the new response fields remains a post-deployment check.


### Common human-transfer history source delivery — 2026-11-25

The user explicitly requested committing/pushing the API on master and updating/pushing the parent
on main with documentation and the submodule pointer. API commit
[394caddfd5acd9b7dda2c53dd0e69c29d21e9a44](https://github.com/red-abacus/nexcall-api/commit/394caddfd5acd9b7dda2c53dd0e69c29d21e9a44)
contains the verified shared contract, collectors, migration 041, regression checks and web handoff.
Parent main records the decision history and points to that API commit. Unrelated web changes are
excluded, and no feature branch was created.

The API push queued its existing
[production workflow 45619130718](https://github.com/red-abacus/nexcall-api/actions/runs/45619130718).
No successful deployment, production migration or live response-field verification is inferred from
this push. Verify workflow success, application readiness, migration 041 and both history endpoints
after rollout; web-client integration remains separate.


## AGENT-011: Inbound interaction lifecycle through human handover

- **Decision/source date:** 2026-11-25.
- **Origin:** Explicit user requirement for one interaction that remains active through transfer;
  technical choice to reuse the existing scoped socket hint and store original-caller state separately.
- **Status:** Implemented in the local API checkout; verification recorded below.
  No commit, push, deployment or live verification is implied.

Previously the Vapi AI leg ending completed the interaction, and direct JDBC human-transfer changes
were not announced through callTaskUpdates. The user requested a transferring state, return to active
progress after acceptance, and completion only when the caller's conversation ends.

The inbound interaction now follows inProgress → transferring (request/dial/screen) → inProgress
(operator acceptance) → completed (original caller end). Keep the same interaction and call-detail
IDs. Failure/refusal announcements remain inProgress until caller hang-up; caller completion wins
over a later transfer callback. AI/provider attempt status, time, recording and billing remain separate.
Shared humanTransfer retains AGENT-010's five fields and evidence rules: acceptedAt alone is not
confirmed bridging; connected still requires the signed final DialBridged evidence for this route.

Additive migration 042 records caller_status on the admitted route receipt. Save active state only
when unset during transfer intent, then terminal state from the verified original Twilio CallSid /
CallStatus callback or existing carrier reconciliation. A Vapi/SIP leg end cannot complete an active
managed caller. Historical transfers without caller tracking retain their saved outcome.

Reuse /user/task-events/callTaskUpdates with its existing three-ID payload. Publish after commit
for transfer intent, uncertain redirect, acceptance/refusal, final outcome, caller end and interaction
JSON changes. Coalesce duplicates within a transaction, emit none on rollback, preserve existing
campaign/tenant/session authorization. Clients refetch interactions and update the same row. Socket
delivery remains best effort, with refetch on open/reconnect. This extends AGENT-010's initial
notification behavior without introducing another topic or adding transcript/provider details.

Sources: explicit user request in this conversation, inspected original Twilio StatusCallback and
Vapi webhook paths, existing after-commit notification collector. Implementation plan:
[backend lifecycle notes](../../apps/api/docs/superpowers/plans/2026-11-24-inbound-sip-transfer-validation.md#inbound-interaction-lifecycle-and-socket-hints--2026-11-25).
Client contract: [human-transfer handoff](../../apps/api/docs/api/human-transfer-web-handoff.md#inbound-adaptation).

Open limits: accepted is not a live bridge-confirmation event; missing transfer outcome callbacks
can still leave humanTransfer unknown/in-progress. EN and live lifecycle/audio checks after rollout
remain separate. No recording, human billing, outbound route or client source change is included.


Verification, 2026-11-25: MySQL-enabled `./gradlew build` passed 1,352 tests total,
1,351 passed, zero failures/errors, 294 MySQL tests and one optional live-provider test skipped.
After the last caller-state projection/notification adjustments and expanded real-MySQL late-Vapi
callback regression, the focused suite passed all 158 tests (90 MySQL), zero failures/errors/skips,
and bootJar passed again. It includes transfer control/history, commit/rollback notification
semantics, existing scoped socket contracts and real-migration callback ordering checks.
The temporary external 2 GB test heap is reused without project toolchain/configuration changes.
Liquibase XML parsing, diff whitespace and local handoff/decision link checks pass.
No commit, push, deployment, provider change or live call has been performed for AGENT-011.


2026-11-25 follow-up: [CALL-019](2026-11-18-campaigns-and-calling.md#call-019-inbound-campaign-recording-uses-the-original-twilio-call)
replaces the earlier AI-only inbound recording limitation with opt-in whole-call Twilio capture in
the local checkout. It leaves the shared handover contract and AI usage measurement unchanged;
profile refresh, deployment and live continuity verification remain separate.

[SUB-023](2026-11-18-subscriptions-and-call-limits.md#sub-023-agent-started-calls-use-ai-minutes-through-caller-hang-up) resolves the human-leg billing decision for newly admitted agent calls.

### AGENT-012 Inbound caller hears the campaign voice before operator dial

**Source and date** — Explicit user decision, 2026-11-25. The Twilio caller announcement introduced a voice change at handover. This replaces the caller-announcement portion of AGENT-008.

**Status** — Pushed to API master in `4e0cd08`. The shared operator introduction ends after the accept-or-decline instruction in both languages. The inbound Vapi function speaks the shared RO/EN caller message with blocking enabled before it calls NexCall. NexCall then redirects the original caller directly to Twilio Dial. Twilio continues screening the operator and delivering the no-answer message. Deploy the API and refresh existing inbound assistant profiles together. Live verification should confirm the caller hears the whole phrase once before the operator rings.

[Vapi blocking tool messages](https://docs.vapi.ai/whats-new/2025/2/4) · [Inbound handover plan](../../apps/api/docs/superpowers/plans/2026-11-24-inbound-sip-transfer-validation.md).

### AGENT-013 Scripted destinations for inbound routing

**Source and date** — Explicit user request, 2026-11-26, for the inbound routing pilot. Vapi selects
the payment or appointment destination from the campaign script and passes the number to NexCall.
NexCall uses it directly for the Twilio dial without finding a contact.

**Status** — Implemented in the API checkout. The full MySQL-enabled Gradle run and bootJar passed (1420 passed, 1 skipped), including all 34 inbound transfer MySQL cases with mocked providers. API master `67b94a8` reached production on 2026-11-26 with successful deployment and post-deployment smoke checks. Campaign 77 was subsequently provisioned and activated for the live pilot. [Deployment run](https://github.com/red-abacus/nexcall-api/actions/runs/45843039362)

**Technical choices and consequences** — Extend AGENT-008 and AGENT-012 with an optional
`destinationNumber` on the existing tool. For inbound ROUTING only, derive the allowed numbers
from contiguous international numbers in the applied campaign instructions. Only include intended
destinations in that format. Omitted arguments retain general handover. Signed call correlation,
duplicate protection, operator screening, history and caller-leg handling continue on the same path.
Saved assistants need reprovisioning. On 2026-11-26, production campaign 77 dialed the scripted payment destination (ending 7350), distinct from its general handover number. The transfer reached Twilio screening and ended with persisted outcome `cancelled` and no acceptance timestamp. This verifies payment destination selection and dialing. A subsequent campaign 77 transfer to the appointment number (ending 7177) recorded CONNECTED/connected, confirming acceptance and bridging. Audio quality and how often the agent selects the intended route across varied requests remain unverified. See the [API contract and pilot instructions](../../apps/api/docs/api/shared-agent-configuration-web-handoff.md#prompt-and-handover-behavior).

### AGENT-014 Creation-time CALL presets and campaign-authored instructions

**Explicit user decision, 2026-11-26. Pushed to API master in `58d85ca`.** Seed 16 immutable shared CALL presets in the database across four purposes, Romanian and English, and inbound and outbound directions. Remove industry from inbound and outbound campaign configuration and from all preset records, validation and selection filters. At creation the client copies the preset into editable campaign fields. The saved campaign owns its instructions independently of the preset.

This replaces the automatic foundation, industry, purpose and direction prompt paragraphs for CALL agents. Use the resolved campaign guidance, handover condition and selected information to collect. Send the opening separately. The rollout requires no campaign data migration or legacy prompt branch. Previously provisioned assistants can retain their provider configuration until normally reprovisioned.

Source: the supplied presets backend contract and the user's clarifications on 2026-11-26. The user superseded the handoff’s industry requirement and confirmed AGENT-013 scripted destinations for inbound ROUTING. The Romanian and English inbound routing presets should illustrate separate billing and appointment destinations and ask for clarification when the caller’s intent is unclear. The user edits the copied script or supplies the actual numbers. Retain the general transfer number for ordinary human handover. The initial implementation resolved inbound static fields. The user superseded that approach on 2026-11-26. Inbound presets use manual editing markers. Users replace or remove them in the copied script before saving. NexCall derives allowed destinations directly from final instructions. Only outbound supports per-recipient substitution. Migration 049 installs the catalog. The [preset integration handoff](../../apps/api/docs/api/settings-scripts-web-handoff.md) documents the paged picker response and creation payloads. Local MySQL tests verified repeatable installation of exactly 16 presets and persistence of the editable routing configuration through provisioning. The full API build passed (1,338 tests passed, one live voice-preview test skipped). Deployment and client integration are pending.

[SETTINGS-001](2026-11-26-settings-usage.md) sets tenant preset visibility to tenant-owned records plus the 16 built-ins.
