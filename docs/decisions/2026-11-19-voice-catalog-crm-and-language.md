# Voice catalog, CRM export and campaign language

Recorded on 2026-11-19 (Europe/Bucharest). This records the user's API-only follow-up to
`/Users/folea/Downloads/inbound-backend-fixes-handoff.md` and the subsequent API review in this
chat. The user approved the API design, plans and API-only implementation on 2026-11-19.
CRM export, language and conflict changes are implemented in the checkout; local verification and deployment
are separate facts. No deployment, provider changes or client implementation were performed.

### VOICE-001 Replace the hardcoded inbound voice list

**Status:** Accepted design · Explicit user request and live-approach approval; implementation externally pending.

List voices dynamically from Vapi instead of maintaining the four hardcoded choices. Pagination
is allowed. Voice listing and create/update validation must use the same source and preserve
existing saved configurations. The user explicitly confirmed the **entire available library**:
all voices available to the configured Vapi organization across Vapi and third-party providers,
not only native Vapi Voices. The provider/language should be filterable without restricting the
catalog to the current four choices. A supported catalog API/source, language metadata and preview
availability remain to be resolved.
Do not claim the entire dashboard library is exposed by a verified public API yet.

The user selected the **live catalog approach** over a periodically synchronized NexCall copy.
Read-only upstream requests are the chosen direction; no persistent catalog or catalog cache is
approved. The [approved API design](../../apps/api/docs/superpowers/specs/2026-11-19-inbound-workflow-completion-design.md)
records the prerequisite to verify supported organization-level
server-key access. Inspection of the public dashboard bundle found a per-provider voice-library
route, but authenticated access and its supported contract are not yet verified.

Context: the current catalog is `InboundAgentCatalog`; the existing integration is `VapiClient`.
Vapi distinguishes its native voices from its organization-visible multi-provider Voice Library.
Sources: [voice overview](https://docs.vapi.ai/providers/voice/overview),
[public API index](https://docs.vapi.ai/llms.txt).

The user confirmed credentials are unavailable in this environment. The [catalog verification record](../../apps/api/docs/api/vapi-live-voice-catalog-contract.md) preserves the exact prerequisite. No guessed production adapter was added; the existing hardcoded catalog remains until that gate passes.

### CRM-025 Reviewed inbound contact export and existing-contact history

**Status:** Implemented in checkout · Explicit user requirement and approved design; focused MySQL/HTTP verification passed; not deployed.

Extend Save to CRM with optional reviewed contact fields for a single task while preserving
existing batch requests. Validate names, phones and email; retain tenant/role, CRM feature and
active-call checks. Create or link the contact and import history transactionally without
rewriting original task facts. Reuse an existing matching contact within the authorized tenant,
preserve its profile and import the new history with source-identity deduplication on retries.
Generic inbound task editing remains forbidden.

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
Selected Deepgram, AssemblyAI English, unversioned Vapi English and ElevenLabs adapters are enumerated in the
[language contract/rollout handoff](../../apps/api/docs/api/campaign-language-web-handoff.md). Undeclared/unknown combinations
fail closed; these local fixtures are not live provider or pronunciation validation. Config-less voice drafts remain
editable, but completing configuration/start/reservation requires compatible declared speech.
Migration 036 HALTs before adding language if any non-deleted outbound rows remain, including finished rows.
Drain old API campaign writers before retiring/resetting campaigns. Migration 036 adds language and its CHECK in a single atomic MySQL ALTER, so a racing old-row insert cannot leave a partially applied schema.
No data reset or default/backfill was performed.

Sources: handoff section 4 and the user's explicit follow-up;
[guide policy](2026-11-18-campaigns-and-calling.md#call-011-shared-guide-and-reusable-script-snapshots).

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
its production adapter remains conditional. The other deliverables are independent of that gate.
CRM, activation conflicts and campaign language have passed focused API and disposable-MySQL checks.
Full verification 2026-11-19: 755 tests passed with zero failures/errors/skips, including 132 real MySQL tests; `bootJar` passed. One fresh final review identified three P2 issues; deterministic MySQL regressions reproduced them and the single fix pass passed the full suite. The live catalog remains pending authenticated supported access.
No commits, branches, data reset, live provider pilot, client edits or deployment are part of this delivery.

[Implementation report, verification and technical rulings](../../apps/api/docs/superpowers/reports/2026-11-19-inbound-workflow-completion.md).
