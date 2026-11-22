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
