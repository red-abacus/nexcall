# Voice catalog, CRM export and campaign language

2026-11-22 follow-up: [AGENT-003](2026-11-22-shared-agent-configuration.md#agent-003-shared-agent-catalogue-configuration-and-vapi-warm-transfer)
moves the catalogue to `/api/agents/voices` and adds common nested agent configuration. The earlier
VOICE/CALL records below preserve their source dates and contracts; use the newer handoff for current routes.

Recorded on 2026-11-19 (Europe/Bucharest). This records the user's API-only follow-up to
`/Users/folea/Downloads/inbound-backend-fixes-handoff.md` and the subsequent API review in this
chat. The user approved the API design, plans and API-only implementation on 2026-11-19.
CRM export, language and conflict changes are implemented in the checkout; local verification and deployment
are separate facts. No deployment, provider changes or client implementation were performed.

### VOICE-001 Replace the hardcoded inbound voice list

**Status:** Shared catalog implemented under CALL-017; inbound migration implemented under VOICE-005 (2026-11-22).
**Origin:** Explicit user request and live-source approach approval, 2026-11-19.

Replace the four hardcoded inbound choices with the configured Vapi organization's entire available
multi-provider library. The user chose live reads rather than a synchronized or cached NexCall copy.
The later [CALL-017](#call-017-campaign-voice-selection-and-previews) campaign feature owns the shared
catalog and preview adapter. Its source gate passed after the user supplied a local test connection;
read-only authenticated paging covered 3,848 rows across 21 provider values. The route and callable
slug mapping were verified against Vapi's official client and live responses. Public Swagger still
omits the route, and published version/stability guarantees remain absent. Explicit integration
errors and transport fixtures guard drift; no runtime dashboard scraper is used.

The shared `/api/voices` and preview endpoints are implemented for outbound campaigns. Earlier
inbound adapter/interface sketches are superseded by this shared infrastructure. The subsequent
VOICE-005 migration covers inbound desired/applied configuration and removes the old catalog route,
initially retaining legacy saved-voice compatibility. VOICE-006 subsequently removes that
compatibility by explicit user instruction. Frontend integration remains separate.

Earlier public-source investigation and the initial lack of test credentials are retained in the
[verification record](../../apps/api/docs/api/vapi-live-voice-catalog-contract.md); they describe a
historical gate, not the current implementation status. The test account had no private/custom rows;
private visibility and two-tenant sharing are fixture-verified, not live-private-voice proof. Missing
and denied previews are accepted under [VOICE-003](#voice-003-preview-availability-does-not-block-voice-selection).

Sources: explicit user approvals in this chat;
[voice overview](https://docs.vapi.ai/providers/voice/overview),
[shared API plan](../../apps/api/docs/superpowers/plans/2026-11-19-campaign-voice-selection.md),
[frontend handoff](../../apps/api/docs/api/campaign-voice-selection-web-handoff.md).

### VOICE-005 Inbound uses the shared voice catalog

**Decision/source date:** 2026-11-22. **Origin:** Explicit user request and implementation authorization.
**Status:** Implemented in checkout; verification below is separate from deployment.

Remove `/api/inbound/voices`; inbound and outbound consume `/api/voices` and `/api/voices/preview`.
This completes VOICE-001's inbound catalog migration. Inbound saves the same exact `voiceSelection`
identity, validated by the shared resolver. Unsupported/language-incompatible choices reject;
preview availability does not gate selection. Preserve inbound model/transcriber and provisioning.

Historical technical compatibility choice, superseded by VOICE-006: existing legacy specs retain their exact voice/provider until explicitly
replaced, including unchanged full PUT. New agents require shared selection. No automatic backfill,
default substitution or provider mutation occurs during migration. JSON storage is additive.
Resolution occurs outside DB transactions at save and again before provider provisioning, so catalog
availability affects both; full PUT of shared selections requires live validation. Internal resolution
is callable by durable workers without a user session; list/preview HTTP access remains role-protected.
Older API/workers must be drained before selection-only writes. Client cutover and deployment remain
separate; no web source was changed. This does not unify other agent properties or approve new purposes.

Source: this conversation; [contract delta](../../apps/api/docs/api/inbound-shared-voices-web-handoff.md).

Verification on 2026-11-22: full `./gradlew test bootJar` with disposable local MySQL enabled
reported 832 tests: 831 passed, zero failures/errors, one opt-in live Vapi catalog test skipped.
The new tests cover shared payload identity and rejection, legacy preservation, removed endpoint,
Jackson 2/3 round trips and real MySQL desired/applied/replacement settings. No live provider
mutation, call pilot, client build or deployment was performed.

### VOICE-006 Remove inbound legacy voice compatibility

**Decision/source date:** 2026-11-22. **Origin:** Explicit user instruction that backwards
compatibility is unnecessary and legacy voices can be removed.
**Status:** Implemented in checkout; no deployment or remote data inspection performed.

Remove `LEGACY_VOICES`, legacy lookup/provisioning, the top-level inbound `voiceId` property,
the old constructor and the update exception for unchanged legacy settings. Require the shared
`voiceSelection` on inbound create and full update. Missing selections fail before provisioning;
no automatic conversion, backfill, substitute voice or data deletion is performed.
This supersedes VOICE-005's initial compatibility precaution, retaining its shared catalog design.

Read-only local inspection found zero inbound campaigns and zero `inbound_agent_config` rows.
This establishes local state only; any legacy remote configuration is intentionally unsupported
by this decision and must receive an explicit shared selection before provisioning.
Sources: this conversation, local MySQL counts, [current client contract](../../apps/api/docs/api/inbound-shared-voices-web-handoff.md).

Verification after compatibility removal: MySQL-enabled full `./gradlew test bootJar` passed
on 2026-11-22: 831 passed, zero failures/errors, one opt-in live Vapi test skipped (832 total).
Required selection, missing-selection rejection, exact shared payloads and JSON persistence
are covered. No deployment or live provider mutation was performed.

### VOICE-002 Share private and custom voices with all tenants

**Source/decision date:** 2026-11-19. **Status:** Accepted requirement · Explicit user decision;
implemented in the API checkout. Authenticated public-library access is verified; the test account
had no private rows, so private/custom sharing is verified with synthetic HTTP/campaign fixtures.

The user answered the catalog audience question: **"Share private/custom voices with all tenants."**
ADMIN and CLIENT_ADMIN users from every NexCall tenant may browse, preview and select the same
configured Vapi organization library, including organization-private/custom voices. This resolves
the earlier pending ADMIN-only versus all-tenant choice raised by builder review.

Context: Vapi organization visibility does not itself determine NexCall tenant authorization;
the user explicitly chose a shared platform voice library. Consequences: use the same audience
rule in catalog listing, previews, selection/save/start validation and saved voice scope. Do not
introduce per-tenant voice ownership or sharing administration for this feature. Campaign access
remains tenant-scoped, and credentials/operational provider metadata remain server-side. Test
ADMIN and CLIENT_ADMIN users from at least two tenants using the same private/custom voice while
preserving cross-tenant campaign denial. Sharing approval does not verify vendor access, preview
coverage or provider compatibility, and does not authorize voice cloning/import/generation.

Sources: explicit question reply in this chat;
[API design](../../apps/api/docs/superpowers/specs/2026-11-19-campaign-voice-selection-design.md#3-shared-catalog-and-preview-api),
[implementation plan](../../apps/api/docs/superpowers/plans/2026-11-19-campaign-voice-selection.md),
[CALL-017](#call-017-campaign-voice-selection-and-previews), [VOICE-001](#voice-001-replace-the-hardcoded-inbound-voice-list).

### VOICE-003 Preview availability does not block voice selection

**Source/decision date:** 2026-11-19. **Status:** Accepted requirement · Explicit user answer during
authenticated implementation verification; implemented and locally verified in the API checkout; not deployed.

After private-key access and full paging were verified, the catalog returned 3,848 rows across 21
provider values, of which 270 advertised a sample URL. Sampled Vapi S3 recordings returned 403
AccessDenied; three ElevenLabs samples were retrievable as MP3 bytes under a text/plain header.
No voice licensing/synthesis-rights failure was established by those media errors.

The user explicitly chose **"Yes—continue with available previews"**. A supported voice can be
selected even when its recording is absent or inaccessible. Expose preview metadata honestly,
serve working audio through an authenticated bounded endpoint, and report failed/missing samples
as unavailable. Preview metadata advertises a source; permission/availability is established when
audio is fetched. Do not promise that an advertised URL guarantees playback, invent a sample,
silently use another voice's clip, or generate chargeable audio as a GET side effect.

Consequences: the earlier all-preview integration gate is relaxed only for audio availability;
authoritative catalog identity, provider/model/language validation, secret handling, tenant rules
and explicit-voice/no-substitution behavior remain required. Keep the upstream access limitation
visible in the final frontend handoff. The same private Vapi API key/org configuration is reused;
no new production credential is required by this decision.

Sources: user's explicit reply in this chat;
[authenticated verification](../../apps/api/docs/api/vapi-live-voice-catalog-contract.md#authenticated-verification-and-accepted-preview-scope--2026-11-19),
[campaign voice plan](../../apps/api/docs/superpowers/plans/2026-11-19-campaign-voice-selection.md).

### CRM-025 Reviewed inbound contact export and existing-contact history

**Status:** Implemented in checkout · Explicit user requirement and approved design; focused MySQL/HTTP verification passed; not deployed.

Extend Save to CRM with optional reviewed contact fields for a single task while preserving
existing batch requests. Validate names, phones and email; retain tenant/role, CRM feature and
active-call checks. Create or link the contact and import history transactionally without
rewriting original task facts. Reuse an existing matching contact within the authorized tenant,
preserve its profile and import the new history with source-identity deduplication on retries.
Generic inbound task editing remains forbidden. [CRM-026](#crm-026-inbound-interaction-contact-and-timeline-status) extends this flow to automatically matched agent interactions and exposes timeline status.

The optional reviewed contact is supported for exactly one task. Successful reuse is returned in
`savedTaskIds`; multiple matched profiles fail with `pages.crm.errors.ambiguousContactPhones`.
Existing batches preserve their behavior. Active-call checks also apply to already-linked reviewed exports.

Implementation technical choice: inbound-agent caller tasks use the existing server-managed `source=inbound`
classification. Migration 035 backfills that classification and exempts those tasks from the existing outbound
campaign/contact uniqueness index, allowing different caller identities to preserve separate histories while
linking one profile. Outbound uniqueness remains enforced. Phone-writing transactions acquire the tenant
before contact locks to prevent export/phone-edit deadlocks. Both legacy and reviewed exports lock tenant before campaign/task; reviewed reuse locks contacts before campaign/task and reads current links after the tenant guard, avoiding both lock inversion and stale concurrent retries. [CRM contract and evidence](../../apps/api/docs/api/reviewed-crm-export-web-handoff.md).

Sources: handoff section 3 and the user's explicit follow-up; [existing history policy](2026-11-18-campaigns-and-calling.md#call-002-disposition-identity-and-crm-history),
[request](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/dto/SaveCallTasksToCrmRequest.java),
[export](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CallTaskService.java).

### CALL-013 Explicit outbound campaign language

**Status:** Implemented in checkout · Explicit user requirement and approved design; focused contract/migration/request tests passed; reset and deployment pending.

Persist an explicit campaign language for operator and AI outbound campaigns. Validate supported
codes, expose the choices, return the language in list/detail/context responses, preserve PATCH
omission and cloning, and apply it to AI speech/transcription and script context. Never infer it
from a user's UI locale or silently translate/overwrite existing scripts. Validate compatibility
with configured voices/transcribers and make campaign retirement/reset an explicit migration
prerequisite, without an old-client language default.

Require an explicit `en` or `ro` for every newly created outbound campaign. The user confirmed
there is no production compatibility requirement and existing campaigns may be retired/recreated.
Do not introduce an inferred/default language or executable null-language fallback. PATCH omission
preserves the saved language; null/blank/unsupported values are invalid. Retiring/resetting data
is a documented migration prerequisite, not authorization for the agent to delete campaigns.

The language plan declares supported language capabilities through NexCall-only metadata in the
existing speech preset configuration, strips it from provider payloads and rejects undeclared or
incompatible combinations. The technical choice is implemented with provider-specific request-shape tests and primary documentation references.
The later [CALL-016](#call-016-language-aware-outbound-speech-fallbacks) extension tries an explicitly
configured language fallback for an incompatible speech component before rejecting; required
campaign language and provider compatibility checks remain intact.
Selected Deepgram, AssemblyAI English, unversioned Vapi English and ElevenLabs adapters are enumerated in the
[language contract/rollout handoff](../../apps/api/docs/api/campaign-language-web-handoff.md). Undeclared/unknown combinations
fail closed; these local fixtures are not live provider or pronunciation validation. Config-less voice drafts remain
editable, but completing configuration/start/reservation requires compatible declared speech.
Migration 036 HALTs before adding language if any non-deleted outbound rows remain, including finished rows.
Drain old API campaign writers before retiring/resetting campaigns. Migration 036 adds language and its CHECK in a single atomic MySQL ALTER, so a racing old-row insert cannot leave a partially applied schema.
No data reset or default/backfill was performed.

Sources: handoff section 4 and the user's explicit follow-up;
[guide policy](2026-11-18-campaigns-and-calling.md#call-011-shared-guide-and-reusable-script-snapshots).

### CALL-014 Default bilingual outbound speech presets

**Source/decision date:** 2026-11-19. **Status:** Superseded before deployment by [CALL-015](#call-015-preserve-previous-english-speech-defaults).
The bilingual configuration was implemented and locally checked, then replaced at the user's request.
The fallback/override mechanism below is retained; its original preset selection is historical.

Original source: explicit user request
to make the preceding English/Romanian preset JSON the fallback when the variable/environment value
is absent. Local configuration verification is recorded below; production deployment is separate.

Use `test` IDs for Deepgram `nova-3`, OpenAI `gpt-4o`, and ElevenLabs `eleven_flash_v2_5` with
`voiceId: sarah`. Speech presets declare `nexcallLanguageCodes: "en,ro"`. Campaign language remains
explicit under CALL-013; this does not introduce a campaign-language default or relax compatibility checks.

Technical choice: keep one canonical default in the API Helm chart's
[vapi-presets.json](../../apps/api/deploy/helm/nexcall-api/files/vapi-presets.json), loaded when the
entire `vapi.presets` value is unset/empty. Resolve the fallback in the template rather than deep-merging
it through chart values, so a custom provider does not inherit incompatible model/voice fields.
Both the parent and standalone API deployment workflows accept `NEXCALL_VAPI_PRESETS_JSON` from
GitHub Actions variables or workflow/job environment; non-empty explicit JSON takes precedence.
Malformed explicit JSON fails deployment. Empty variable/environment values preserve a values-file
catalog or use the chart fallback. This replaces the standalone workflow's former Flux/Nico English-only
fallback and the empty effective Helm catalog; explicit existing catalogs remain authoritative.

Credentials, dispatch enablement and the saved platform preset selection remain separately required.
A saved `test`/`test`/`test` selection stays valid. Future reservations adopt the deployed preset settings;
existing request snapshots remain unchanged. No automatic provider resource mutation, voice pilot,
platform-selection write or deployment is performed by this task.

Sources: the user's default request in this chat; [configuration/selection contract](../../apps/api/docs/api/campaign-language-web-handoff.md#existing-speech-preset-configuration),
[deployment configuration](../../apps/api/docs/deployment-gke.md#vapi-rollout).

Local verification for CALL-014: Helm lint passed with base defaults and production dispatch enabled;
rendered omitted/empty/null catalogs use the default, while custom catalogs with the same or different
IDs remain exact. The packaged chart renders the fallback. Each real deployment shell block was
executed against local Helm rendering in six cases (unset, empty, two custom catalogs, malformed JSON,
and partial configuration); no cluster/provider request was made. Nine existing deployment/secret/
language-policy checks passed in a fresh Gradle run, zero failures/errors/skips. Full API/MySQL checks
were not repeated for this configuration-only change; the 755-test report below remains historical.

### CALL-015 Preserve previous English speech defaults

**Source/decision date:** 2026-11-19. **Status:** Implemented in checkout · Explicit user correction:
restore the previously used configurations as the defaults. Supersedes CALL-014's bilingual preset
selection; retains its canonical fallback file and explicit-override behavior. Deployment remains separate.

The later [CALL-016](#call-016-language-aware-outbound-speech-fallbacks) extends this with Romanian
fallbacks. English identity remains unchanged; the Romanian-rejection consequence below is historical
for catalogs without a compatible configured fallback.

Default `test` IDs now use Deepgram `flux-general-en` with `language: en`, OpenAI `gpt-4o`, and
Vapi `Nico`. Transcriber and voice declare `nexcallLanguageCodes: "en"` for the current API validator.
This preserves the previous provider/model/voice identities. No `version` or voice `language` field
is introduced. Both deployment workflows use this catalog when the variable/environment and Helm
catalog are unset/empty; explicit non-empty catalogs remain authoritative and are never merged
with fallback provider fields.

Consequence: the default supports English. Romanian campaigns still fail
`general.messages.400.campaign.voiceLanguageUnsupported` until a compatible custom transcriber/voice
catalog is selected. Do not label English-only presets as Romanian-compatible. Existing credentials,
dispatch enablement, platform selection and immutable saved request behavior are unchanged. This
does not prevent configuring a separate multilingual catalog explicitly.

Sources: user's request in this chat; [current default JSON](../../apps/api/deploy/helm/nexcall-api/files/vapi-presets.json),
[language/configuration handoff](../../apps/api/docs/api/campaign-language-web-handoff.md#existing-speech-preset-configuration),
[Vapi Deepgram matrix](https://docs.vapi.ai/providers/transcriber/deepgram),
[current Vapi voice catalog](https://docs.vapi.ai/providers/voice/vapi-voices). Provider documentation
still lists Nico and Flux English; live account access/call quality is not verified by this local change.

Local verification for CALL-015: both Helm lint modes passed; omitted/empty/null configurations and
the packaged chart render the restored English defaults. Twelve execution cases across the actual
two deployment shell blocks passed, preserving explicit custom catalogs and malformed-JSON failures.
Nine existing workflow/secret/language-policy checks passed in a fresh Gradle run, zero failures,
errors or skips. No deployment or live provider call was performed.

### CALL-016 Language-aware outbound speech fallbacks

**Source/decision date:** 2026-11-19. **Status:** Implemented and locally verified in checkout; not deployed.
**Origin:** Explicit user request to add a Romanian fallback and explain web voice selection. Component-level
resolution and additive configuration shape are technical implementation choices under that request.

Keep the previous English Flux/GPT-4o/Nico identity. Add Romanian speech presets (Deepgram Nova 3,
ElevenLabs Flash v2.5/Sarah) plus `languageFallbacks.en`/`.ro` transcriber/voice references in the built-in
catalog. Prefer each compatible platform-selected speech component. Only an incompatible component
uses its configured fallback; preserve the selected language model. Missing or invalid fallback/provider
capabilities still reject. Explicit custom catalogs do not inherit built-in fallback references.

Use the same provider policy for campaign validation and request snapshots. Strip capability metadata
before provider submission. Reserved speech/model/language instructions remain fixed; existing
recording/duration controls can still refresh before submission. Keep credentials, flags, required
campaign language and the existing saved platform selection; no model selection is silently created.
Platform GET adds a `languageFallbacks` map; PUT remains the existing three-preset-ID contract.

Helm rendering rejects malformed fallback shapes, even with dispatch disabled. For direct Spring
configuration, malformed typed shapes fail startup consistently with existing catalog binding.
Well-formed but unusable references retain voiceLanguageUnsupported when needed.

The existing web contracts differ: inbound agents save a per-agent voice ID, while outbound campaigns
use platform-wide settings. The full live library remains pending under VOICE-001. That fallback
delivery did not implement per-campaign outbound voice selection; the user subsequently confirmed
the requirement under CALL-017. No frontend source was modified by the fallback task.

This extends CALL-015 and supersedes its Romanian rejection when compatible configured fallbacks
are present. It does not reinstate CALL-014's replacement of the English voice. Provider documentation
supports the chosen Romanian request shapes; live account access/pronunciation are not verified.

Sources: this chat; [design](../../apps/api/docs/superpowers/specs/2026-11-19-language-aware-speech-fallbacks-design.md),
[plan and verification](../../apps/api/docs/superpowers/plans/2026-11-19-language-aware-speech-fallbacks.md),
[frontend contract delta](../../apps/api/docs/api/language-aware-speech-fallbacks-web-handoff.md),
[Deepgram languages](https://docs.vapi.ai/providers/transcriber/deepgram),
[ElevenLabs languages](https://docs.vapi.ai/providers/voice/elevenlabs).

Final verification for CALL-016: 767 tests passed with zero failures/errors/skips, including real
MySQL integration suites; bootJar passed. One independent review and a single fix pass addressed
configuration-shape rejection, accurate snapshot wording and reverse-English regression coverage.
Actual packaged Helm/default/custom workflow consumption and generated OpenAPI were verified locally.
No provider pilot, deployment, frontend source change or Git commit was performed.

### CALL-017 Campaign voice selection and previews

**Source/decision date:** 2026-11-19. **Status:** Implemented and locally verified in the API checkout. **Origin:** Explicit user clarification, approved API design/plan and implementation approval.
No frontend implementation, deployment or provider-resource mutation is included.

Keep presets and English/Romanian speech fallbacks as defaults. `COLD_CALLING` campaigns in
`VOICE_AGENT` mode can browse the shared library, retrieve available samples and save an optional
`voiceSelection`. The user chose **default unless overridden**. An explicit identity is honored or
fails descriptively; it never falls back to another voice. Model/transcriber remain platform defaults,
and an unused missing/incompatible default voice cannot block an explicit override.

Implemented contract: authenticated `GET /api/voices` and `GET /api/voices/preview`; optional top-level
campaign identity, PATCH omission/preserve and null/reset, readable identity/name, IDLE duplication
and mode-change clearing. Migration 037 stores nullable server-resolved JSON without backfill. New
choices/language changes/starts are verified outside transactions, followed by final authorization,
allocation and stale-state checks under existing locks. Reads, unchanged edits, pausing and duplication
avoid live voice lookup. Invalid saved state cannot become a default or be rebound to a new Vapi
organization. Explicit reset/replacement and pausing remain recovery paths. Calls use locally resolved
saved configuration; reservation snapshots preserve speech across later edits and leave ongoing calls intact.

The authenticated source gate passed after local credentials were supplied. The user then explicitly
accepted available previews under [VOICE-003](#voice-003-preview-availability-does-not-block-voice-selection).
Both roles share private/custom voices under [VOICE-002](#voice-002-share-private-and-custom-voices-with-all-tenants),
while campaign access remains tenant-scoped. Six provider adapter families have verified configuration
rules; unknown capabilities stay browsable but unselectable. See VOICE-004 for exact identity rules.

Consequences: replace all API/dispatcher instances before enabling the picker. Older dispatchers would
ignore overrides. No new production secret/environment variable is required or retired; existing Vapi
credentials, model/transcriber/default presets and optional `NEXCALL_VAPI_PRESETS_JSON` remain used.
Inbound-agent migration is separate. No live private-voice synthesis or call pilot was performed.
Final API test/bootJar verification passed: 826 tests, zero failures/errors/skips, including 140 real
MySQL tests and one read-only live Vapi adapter check. Generated OpenAPI object/null bodies validate,
and all 192 operation IDs are unique. One independent review and one regression-tested fix pass
addressed all Critical/Important findings. Safe English ProblemDetail messages and stable codes are
implemented; localization is deferred as a
minor final-review finding and is not claimed complete.

Sources: explicit user replies and implementation approval in this chat;
[design](../../apps/api/docs/superpowers/specs/2026-11-19-campaign-voice-selection-design.md),
[implementation plan](../../apps/api/docs/superpowers/plans/2026-11-19-campaign-voice-selection.md),
[frontend handoff](../../apps/api/docs/api/campaign-voice-selection-web-handoff.md),
[verification evidence](../../apps/api/docs/api/vapi-live-voice-catalog-contract.md).

### VOICE-004 Canonical voice identity and bounded live reads

**Source/decision date:** 2026-11-19. **Status:** Implemented and locally verified in checkout; not deployed.
**Origin:** Technical choices within the approved CALL-017 plan, refined by final review; not new user decisions.

Canonical identity is provider + callable slug + resolved model/version. Clients echo the complete
returned identity, including nulls; null is not a wildcard. Equivalent source rows normalize into one
choice; conflicting capabilities for that identity disable it as `VOICE_IDENTITY_AMBIGUOUS`.
Lookup, listing and previews use that same normalization. Native cloned voices require Vapi v2;
model-specific OpenAI TTS voice compatibility is checked instead of treating GPT-Live-only IDs as TTS.
Unknown providers/models/languages fail closed. Saved schema/payload/organization are validated before
retained identity can be refreshed. Prepared write proofs are immutable, privately constructed and
checked against both the requested identity and locked campaign snapshot.

Each request loads complete individual providers transiently before offering choices, to detect
repeated pages/canonical collisions. No persistent cache/mirror is introduced. Limits: 1,000 rows per
upstream page, 20 pages per provider, 32 requests and 15 seconds total; each complete HTTP exchange/body
has at most five seconds or the remaining budget. Oversized/stalled/drifting responses fail explicitly.
Signed cursors bind filters, organization and a stable provider-content fingerprint; changed catalogs
require paging restart. Cost: repeated live reads and upstream latency/rate-limit exposure, accepted
in favor of adding no unapproved cache. A provider exceeding bounds returns an integration error.

Preview sources are restricted to the three verified HTTPS media hosts, with no credentials/redirects,
10 MiB/10-second bounds and audio-signature checks. MP3 requires complete consecutive frames, not an
ID3 tag alone. Provably expired signed URLs are unavailable before fetch; other advertised samples
may still return 404 on denied/missing media. No audio quality, full decoding or synthesis claim is made.
OpenAPI 3.1 uses real object/null unions, and malformed identity JSON returns a stable selection code.
Internal entity name `selectedVoice` avoids public DTO auto-mapping; HTTP Jackson 3 parsing remains
separate from Hibernate's Jackson 2 record mapping.

Sources: final independent review and regression-tested single fix pass in this chat;
[API plan](../../apps/api/docs/superpowers/plans/2026-11-19-campaign-voice-selection.md),
[provider verification](../../apps/api/docs/api/vapi-live-voice-catalog-contract.md),
[OpenAI TTS compatibility](https://developers.openai.com/api/docs/guides/text-to-speech).

### NUM-006 Explain outbound activation conflicts

**Status:** Implemented in checkout · Explicit approved design; focused HTTP and real MySQL allocation/release checks passed; not deployed.

Replace the generic activation `numberAssigned` response with stable, tenant-safe blocker reasons
and permitted recovery actions while retaining atomic allocation checks. Reuse the current
role-scoped association information and preserve outbound sharing, exclusive inbound-agent
reservations, cutover fences and the number release/restart lifecycle. Do not automatically
release an active route or pause/detach an incumbent.

Owned blockers emit `INBOUND_AGENT_RESERVED` / `INBOUND_CUTOVER_IN_PROGRESS`; cutover takes precedence.
Released campaigns preserve their existing ProblemDetail code and gain recovery hints. Missing/foreign inventory
retains indistinguishable existing ineligible/not-found behavior, without emitting hidden blocker reasons.
The proposed missing/unassigned reason names are not exposed for inaccessible resources.
See the [activation contract](../../apps/api/docs/api/number-activation-conflicts-web-handoff.md).

Campaign 49's original live incident is not diagnosed from source alone. Its deployed revision,
number ownership, reservation/cutover state and relevant logs are still needed to classify it.
This change does not alter the approved rejection of non-Collections outbound return calls.

Sources: user's question; [number lifecycle](2026-11-19-number-associations-and-release.md),
[inbound policy](2026-11-18-inbound-routing-and-agents.md#in-009-share-outbound-numbers-within-one-tenant).

## Implementation and verification

The [API plan overview](../../apps/api/docs/superpowers/plans/2026-11-19-inbound-workflow-completion.md)
links four focused plans for live catalog, reviewed CRM export, mandatory campaign language and
activation conflict details. The catalog starts with read-only upstream access/contract verification;
its initial production-adapter gate was subsequently resolved under CALL-017. The other deliverables
were independent of that gate.
CRM, activation conflicts and campaign language have passed focused API and disposable-MySQL checks.
Full verification 2026-11-19: 755 tests passed with zero failures/errors/skips, including 132 real MySQL tests; `bootJar` passed. One fresh final review identified three P2 issues; deterministic MySQL regressions reproduced them and the single fix pass passed the full suite. That historical run preceded the subsequent CALL-017 catalog implementation; its current verification is recorded in the campaign voice plan.
No commits, branches, data reset, live provider pilot, client edits or deployment are part of this delivery.

[Implementation report, verification and technical rulings](../../apps/api/docs/superpowers/reports/2026-11-19-inbound-workflow-completion.md).

### CRM-026 Inbound interaction contact and timeline status

**2026-11-26 · Explicit user requirement · Pushed to API master (`e15d63e`) and web master (`216af2f`).** An inbound interaction must
identify its CRM contact when one exists and tell the client whether that call appears in the
contact's timeline. This extends CRM-025. Campaign 70 exposed the distinction when a manual
export linked the caller task but the interaction was missing from CRM updates.

The API matches a unique active phone contact within the tenant, preserves reviewed links and
uses the individual call details ID for each timeline entry. The list projects current contact
and timeline state independently. Repeating Save to CRM can import existing finished calls.
See the [client contract](../../apps/api/docs/api/inbound-agent-web-handoff.md#interactions-playback-and-crm).

The MySQL-enabled API build passed with 1,405 tests passing and one optional provider test skipped. The inbound web package built and all 80 inbound tests passed. Live verification on campaign 70 is pending.
