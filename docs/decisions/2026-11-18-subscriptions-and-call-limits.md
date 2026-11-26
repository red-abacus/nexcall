# Subscription and call limit decisions

Consolidated on 2026-11-18 (Europe/Bucharest) from the supplied manual-billing handoff,
[implementation plan](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md),
[web differences](../../apps/api/docs/api/subscriptions-web-handoff.md) and API source. All entries
have that source date. “Agreed requirement” refers to the plan's explicit agreed section;
choices originally listed for review are identified separately. Implementation status refers
to the local checkout, not deployment or frontend completion. See the [index](README.md).

### SUB-001 Tenant subscription ownership and access

**Status:** Accepted and implemented · Agreed requirement.

Subscriptions belong to tenants, with separate AI/operator minute pools shared across campaigns.
Platform admins manage commercial operations; benefits supplement existing role, tenant and
environment restrictions. Keep one current base subscription, purchased term snapshots and
outstanding-invoice protection against replacement. Publishing freezes commercial terms;
changed terms originally required a new package identity; [SUB-021](#sub-021-edit-catalogue-packages-independently-of-assigned-subscriptions)
preserved purchased snapshots; [SUB-022](#sub-022-unpublish-after-all-tenants-request-unassignment)
now requires every assignment to request removal before catalogue unpublish/edit.
Archiving preserves prior purchases. [SUB-017](#sub-017-package-unpublish-and-retirement) extends
package lifecycle and supersedes the source plan's allowance for metadata edits after publication.

Evidence: [package service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionPackageService.java),
[billing service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionBillingService.java).

### SUB-002 Manual invoice settlement

**Status:** Accepted and implemented · Agreed requirement and supplied web contract.

Assignment without an eligible trial creates PENDING access and an OPEN invoice.
[SUB-018](#sub-018-package-trials) adds immediate TRIAL access for eligible first-time trials while keeping that invoice unpaid. Only external-payment confirmation
settles the remainder and activates access or grants top-up minutes atomically, once. Derive
admin identity, amount and time on the server; use expected outstanding balance and durable
idempotency fingerprints to reject changed retries. Replays do not restart access or add credit.
There is no automatic charge, client-settable payment status or arbitrary balance editing.

Evidence: [settlement service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionBillingService.java),
[settlement tests](../../apps/api/src/test/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionSettlementTest.java).

### SUB-003 Prices, packs and immutable quotes

**Status:** Accepted and implemented · Agreed requirement and supplied web contract.

Money is integer EUR cents, excluding VAT. Seed AI at €100/month with 200 AI minutes and Operator
at €100/month with 1000 operator minutes, each with one number. Price 100-minute top-ups
progressively by included plus settled purchased minutes, independent of consumption. AI bands
are €20 through 1000 total, €17 through 5000, then €15; Operator bands are €10 through 5000,
€8 through 10000, then €6. Ten AI packs cost €194. Quotes expire after five minutes; ordering
snapshots price without granting minutes, and an open top-up blocks another order/quote.

Evidence: [pricing](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionPricing.java),
[seed migration](../../apps/api/src/main/resources/db/changelog/031-tenant-subscriptions.sql).

### SUB-004 Terms, allowance periods and renewal

**Status:** Accepted and implemented · Agreed requirement and supplied web contract.

The original 1/3/12-month restriction is superseded by [SUB-015](#sub-015-positive-integer-package-price-duration)
on 2026-11-23. Support RECURRING or ONE_TIME and MONTHLY or PER_TERM allowances. Monthly
resets are independent of a quarterly/annual paid term and use the original UTC anniversary,
clamping short months without drifting later anniversaries. Included/top-up credit expires
without rollover. A recurring term produces one renewal invoice and waits for settlement;
one-time access expires. A new paid term starts at settlement. New unpaid subscriptions can
return null paid-term/allowance dates.

Evidence: [period and renewal logic](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionBillingService.java),
[response models](../../apps/api/src/main/java/com/nexcall/monolith/shared/dto/subscription/SubscriptionModels.java).

### SUB-005 Provider-backed second accounting

**Status:** Implemented · Technical choice resolving the plan's billable-duration review item;
separate user approval was not recorded in that item.

Charge connected conversation seconds once per logical customer call. Ringing/failed unanswered
attempts cost zero; connected voicemail counts. Use authenticated provider evidence, not browser
totals or editable feedback/history duration. Replace live estimates with final provider usage;
deduplicate reports and append corrections. AI accounting follows its Vapi call through end or
transfer rather than adding carrier and agent legs together. Any new policy for external legs
continuing after the agent leaves requires a separate decision.

Evidence: [usage service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/CallUsageService.java),
[reconciliation](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/CallUsageReconciliation.java).

### SUB-006 Live metering rather than minute reservations

**Status:** Accepted and implemented · Agreed requirement. **Minute reservations:** Deferred.

New calls require positive settled-plus-live base credit. Admission tracks capacity but does not
reserve seconds. Usage responses include live connected estimates in `*SecondsUsed`, with
`observedAt`, in-progress components and measured overrun. Do not add the in-progress component
twice. An ended AI call awaiting its final report retains an estimate capped at its known end.
Strict pre-reservation budgets remain a future enhancement.

Evidence: [usage projection](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionUsage.java),
[admission](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/CallUsageService.java).

### SUB-007 Pin calls to the admission allowance period

**Status:** Implemented · Technical choice resolving the plan's period-attribution review item.

Preserve the tenant, resource and allowance period chosen at admission through late reports,
corrections and period resets. Tenant concurrency spans periods. This prevents delayed provider
evidence from debiting a new month's credit. User approval was not separately recorded in the
original review item.

Evidence: [session admission and observations](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/CallUsageService.java).

### SUB-008 Shared overrun and recoverable termination

**Status:** Accepted and implemented · Agreed requirement.

Default `minuteOverrunAllowanceMinutes` to five, configurable as a nonnegative integer by platform
admins. [SUB-018](#sub-018-package-trials) excludes trial periods from this paid overrun allowance. It is shared connected time per tenant/resource/admission period, not per call. Block
new calls at base exhaustion; ask providers to end already connected calls when shared extra
time is exhausted. Use database claims, expiring leases and generation fencing for workers in
each pod. Recheck credit before dispatch so top-ups/corrections can cancel obsolete queued
cutoffs. Provider latency can exceed the allowance; retain actual usage. Already dispatched
termination cannot be recalled.

Evidence: [settings](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionPlatformSettings.java),
[worker repository](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionWorkerRepository.java),
[termination](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/CallTerminationService.java).

### SUB-009 Campaign maximum call duration

**Status:** Accepted and implemented · Agreed requirement.

Support optional `maxCallDurationSeconds` from 10 through 14400 for operator Collections/Cold
Calling, outbound agents and inbound agents. Omission preserves a campaign PATCH value; null
clears it. Full inbound PUT carries the desired value. Duplicate the saved setting and snapshot
it per admitted call. Null uses provider defaults, including 600 seconds for Vapi and 7200 for
operator callback Dial; it does not mean unlimited. Changes do not restart active call clocks.

Evidence: [duration validation](../../apps/api/src/main/java/com/nexcall/monolith/shared/phone/CallDurationLimits.java),
[campaign tests](../../apps/api/src/test/java/com/nexcall/monolith/modules/customers/services/CampaignCallDurationTest.java),
[web duration contract](../../apps/api/docs/api/subscriptions-web-handoff.md#campaign-maximum-call-duration).

### SUB-010 Resolve stale wholly unpaid top-ups by audited void

**Status:** Implemented · Technical choice resolving the plan's stale-invoice review item.

Platform admins may void a wholly unpaid TOP_UP invoice when its period has ended or subscription
is inactive, with a reason and idempotency key. Preserve the row as VOID and exclude it from
outstanding totals. Do not void base, paid or partially paid invoices, erase debt, or credit a
later period. Financial resolution of partially paid stale invoices remains deferred; the review
item did not record separate user approval for a broader write-off/refund policy.

Evidence: [void operation](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionBillingService.java),
[web void contract](../../apps/api/docs/api/subscriptions-web-handoff.md#void-stale-unpaid-top-up-invoices).

### SUB-011 Observation mode and enforcement rollout

**Status:** Implemented · Technical choice resolving the plan's rollout review item.

Unassigned existing tenants default to observation. Assignment enables enforcement immediately;
PENDING access blocks new calls. [SUB-018](#sub-018-package-trials) adds eligible TRIAL access despite the unpaid invoice. Base settlement enables enforcement again. Platform admins have
an explicit enforcement recovery toggle; disabled enforcement still measures usage. This is the
current implementation, replacing the draft idea that assignment and enforcement activation
would always be separate steps. Pilot rollout and provider checks remain operational work.

The calling exemption was superseded on 2026-11-26 by [CALL-026](2026-11-18-campaigns-and-calling.md#call-026-pause-a-campaign-before-changing-its-settings-or-deleting-it).

Evidence: [assignment and enforcement](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionBillingService.java),
[rollout guide](../../apps/api/docs/deployment-gke.md#subscriptions-and-call-limits).

### SUB-012 Capacity counts and draft seed defaults

**Status:** Implemented · Technical choices resolving the plan's counts/commercial-default review items.

Count tenant-assigned inventory, active customer memberships, configured non-archived inbound
agents on non-deleted campaigns, and starting/connected/uncertain tenant sessions. Existing voice
dispatch guards remain in addition to subscription concurrency. Seed offers as DRAFT for review;
do not advertise all seed entitlements as approved commercial guarantees. Price a pack crossing
a tier by its ending position. Invoices are initially due on issue; a different payment
deadline policy and further non-promotional offers require a new decision. These defaults are
implementation choices, not newly inferred user approvals.

Evidence: [capacity projection](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionUsage.java),
[pricing](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionPricing.java),
[invoice construction](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionBillingService.java).

### SUB-013 Reuse errors and timestamp contracts

**Status:** Accepted and implemented · Agreed requirement.

Use the existing exception envelope with i18n keys in `message`, not the web mock's `{code}`
shape. Keep ISO technical billing timestamps and existing business-date formats. Add explicit
quota, capacity, idempotency and duration errors. Web must handle these on calling paths as well
as billing screens. Detailed contract deltas remain in the handoff.

Evidence: [response models](../../apps/api/src/main/java/com/nexcall/monolith/shared/dto/subscription/SubscriptionModels.java),
[web error changes](../../apps/api/docs/api/subscriptions-web-handoff.md#error-responses).

### SUB-014 Deferred commercial and client work

**Status:** Deferred · Scope boundary in the supplied contract and plan.

Automated collection, payment-provider checkout, refunds, general replacement and cancellation reversal,
discounts, customer self-service mutations, tax/gross fiscal invoice models, arbitrary credit editing and
minute reservations are outside this delivery. The current ledger is net operational billing.
End-of-period/immediate-unpaid cancellation is added by [SUB-019](#sub-019-tenant-subscription-cancellation).
Read-only current-tenant subscription visibility is added by [SUB-016](#sub-016-current-tenant-subscription-and-access).
Web/mobile integration is separate from the API implementation; a backend record is not proof
that those clients or production provider flows have been delivered.

Sources: [plan deferred extension](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md),
[web contract changes](../../apps/api/docs/api/subscriptions-web-handoff.md).

### SUB-015 Positive integer package price duration

**Source/decision date:** 2026-11-23. **Origin:** Explicit user request and clarification that
one-month packages must remain valid. **Status:** Implemented locally; verification recorded below.
Implemented and verified in the API checkout; deployment is separate.

Allow `PackagePrice.durationMonths` to be any positive integer, including 1 and values above 12,
replacing only SUB-004's fixed 1/3/12 restriction. Keep duration inside each price alongside the
amount for that full term. Existing strict 32-bit JSON integer binding rejects fractions, strings
and overflow; package validation rejects zero/negative values and null binds to an invalid zero.

The billing service already calculates calendar-month terms dynamically. Monthly allowance resets,
PER_TERM allowances, calendar-end clamping, recurring/one-time behavior, prices and currency rules
retain their meanings. Published commercial terms remain locked and tenant term snapshots are
preserved. No schema migration or package rewrite is required. Client integration and deployment
remain separate.

Sources: this conversation; [web contract change](../../apps/api/docs/api/subscriptions-web-handoff.md#package-price-duration--2026-11-23);
[source plan](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md).

Verification, 2026-11-23: MySQL-enabled `./gradlew test --tests '*Subscription*' bootJar` passed
116 tests with zero failures/errors/skips, including 30 MySQL integration tests. New coverage checks
positive durations beyond the former choices, non-positive/malformed JSON rejection, and settlement
of 2- and 13-month terms with MONTHLY/PER_TERM allowance boundaries and calendar-end clamping.
The application JAR built and diff whitespace checks passed. The full application suite was not rerun.

### SUB-016 Current tenant subscription and access

**Source/decision date:** 2026-11-23. **Origin:** Explicit user request to add
`/api/subscriptions/current` after obtaining the JWT, following the proposed separate status/access
lookup. Role/membership resolution, response projection and reuse of lifecycle maintenance are
technical choices. **Status:** Implemented and verified in the API checkout; deployment is separate.

Keep JWT/login contracts unchanged and expose authenticated GET `/api/subscriptions/current`.
Resolve one active, non-deleted tenant membership matching the authenticated tenant user's role;
allow CLIENT_ADMIN, OPERATOR, CLOSER and INTEGRATOR_CRM. Reject missing/ambiguous membership and
platform ADMIN with the existing subscription 403; admins retain their explicit tenant billing
endpoint. No request tenant ID can override identity. Return 200 with null subscription when
unassigned and `Cache-Control: no-store` for authenticated tenant responses.

Expose the purchased package name, identity, selected price ID, included features, allowances,
current status/active flag, term/allowance dates, settled top-up minutes and live usage. Do not
expose invoice/payment records or the catalogue's other prices. Subscription feature gates are
separate: enforcement requires ACTIVE plus the purchased feature (extended to unexpired TRIAL by
[SUB-018](#sub-018-package-trials)); observation/recovery mode
bypasses these gates, including for unassigned tenants. Roles, environment flags, capacities and
minute admission still apply; a true feature gate is not a promise that a call can start.

Reuse admin billing's lazy lifecycle maintenance for renewal, expiry and allowance resets;
repeat reads must not duplicate renewal invoices. This read extension narrows SUB-014's deferred
client scope without authorizing tenant commercial mutations. Frontend/mobile login integration,
production deployment and any future multi-tenant selector remain separate work.

Sources: this conversation; [API handoff](../../apps/api/docs/api/subscriptions-web-handoff.md#current-tenant-subscription-after-login--2026-11-23);
[source plan](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md#current-tenant-read-extension--2026-11-23).

Verification, 2026-11-23: `./gradlew test --tests '*Subscription*' --tests '*TenantAdminLifecycleTest'
bootJar` passed 115 tests; 28 opt-in MySQL tests were skipped in that run. A separate run of
`./gradlew test --tests '*SubscriptionMySqlIntegrationTest' bootJar` with the local MySQL test
configuration passed all 30 tests, zero skipped. Coverage includes pending/active/expired access,
observation mode, live usage, recurring lifecycle refresh without duplicate renewal invoices,
role-matched membership scope, tenant-override isolation, cache headers, and actual JWT/filter-chain
access with Spring-managed customer transactions. Admin billing remains forbidden to tenant users.
The JAR and whitespace checks passed. The full application suite was not rerun.

### SUB-017 Package unpublish and retirement

**Source/decision date:** 2026-11-23. **Origin:** Explicit user request to implement the supplied
`subscription-package-lifecycle-backend-handoff.md` and consolidate subscription handoffs.
Locking-read implementation and rejection of repeated publish/unpublish are technical choices
following its status rules. **Status:** Implemented and verified in the API checkout; deployment
is separate. This supersedes the original source plan's published/archived metadata-edit allowance;
SUB-001 purchased-term immutability and preservation remain in force.

Add platform-admin POST `/api/subscriptions/packages/{id}/unpublish` returning the complete package
as DRAFT only when it is PUBLISHED and no tenant subscription has ever referenced it, regardless
of historical status. That historical-reference restriction is superseded by
[SUB-020](#sub-020-unpublish-after-completed-unassignment), then fully removed by
[SUB-021](#sub-021-edit-catalogue-packages-independently-of-assigned-subscriptions).
All package edits require DRAFT; published/archived metadata-only edits also
reject with 409 `termsLocked`. Publish requires DRAFT and validation. Archive accepts DRAFT or
PUBLISHED, including assigned packages, preserves configuration, and is idempotent when already
ARCHIVED. Archived packages have no restore/edit/delete/publish/unpublish flow. Only unused drafts
may be physically deleted; retain the existing 200 `{}` deletion contract and `deleteDraftOnly` error.

The complete admin list retains all statuses. Assignment choices must filter PUBLISHED, and
server-side assignment rejects withdrawn packages. There is no new tenant purchasing catalogue.
Preview uses existing reads; duplication posts copied configuration to create a new draft identity.
Do not infer archive-menu delivery from the API; the UI source reports that menu action is separate.

Assignment and lifecycle transitions take the same package-row lock. Unpublish's historical
reference query is also a locking/current read so an earlier MySQL repeatable-read snapshot cannot
hide an assignment committed while the lifecycle operation waited. Archival affects no purchased
terms, access, allowances, top-ups, invoices, payments or recurring renewal terms. No schema or
historical-data migration is required.

The attachment's statement that unpublish already existed did not match this checkout; it was
implemented in this task. One self-contained [API handoff](../../apps/api/docs/api/subscriptions-web-handoff.md)
now covers the original billing baseline plus SUB-015/016/017, replacing separate delta/lifecycle
handoffs for integration. Preserve source decisions and the [historical plan](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md#package-lifecycle-extension--2026-11-23).
Sources: this conversation and the supplied UI lifecycle handoff (Downloads attachment).

Verification, 2026-11-23: MySQL-enabled `./gradlew test --tests '*Subscription*'
--tests '*TenantAdminLifecycleTest' bootJar` passed 199 tests, including 42 MySQL integration tests,
zero failures/errors/skips. Added coverage includes complete unused-package edit/publish/unpublish/
delete, historical PENDING/TRIAL/ACTIVE/EXPIRED reference rejection (TRIAL is a test-only historical
status at that verification stage; [SUB-018](#sub-018-package-trials) later implements actual TRIAL access), direct metadata-edit rejection, archival and
repeat archive, archived pending settlement, paid top-ups and recurring renewal, HTTP errors,
and both assignment/withdrawal commit orderings including an earlier repeatable-read snapshot.
The application JAR and diff whitespace checks passed. Full application suite, client flows and
production rollout were not rerun or performed.

### Subscription delivery — 2026-11-23

At the user's explicit request, SUB-015/016/017 and their consolidated API handoff were committed
on API `master` as `4676be4` (`feat(subscriptions): expose tenant access and enforce package lifecycle`).
The parent tracks that API commit and records the decision/documentation changes on its existing
`main` branch, as explicitly confirmed by the user. No new branch was created. The earlier parent
commit `64f6ffe` recording API master integration is also included in this parent delivery.
The unrelated web checkout and local API `.github/.env` remain outside these commits.

Verification remains the 199-test MySQL-enabled subscription/JWT run (42 MySQL tests, zero
failures/errors/skips) and successful bootJar recorded above; no source changes followed that run.
A push is source delivery, not confirmation of deployment, CI success or UI integration.

### SUB-020 Unpublish after completed unassignment

**Source/decision date:** 2026-11-24. **Origin:** Explicit user correction: an unassigned
package must be eligible for unpublish. **Status:** Superseded by
[SUB-021](#sub-021-edit-catalogue-packages-independently-of-assigned-subscriptions) before delivery.
The intermediate implementation was verified locally as recorded below.

SUB-017's historical-reference guard counted completed CANCELLED subscriptions, so preserving
the history required by SUB-019 permanently prevented unpublish. This replaces that part of
SUB-017: ignore CANCELLED history when checking unpublish eligibility. PENDING, TRIAL, ACTIVE
and EXPIRED assignments still block unpublish, including cancellation scheduled for a future
paid/trial deadline. Once cancellation completes for every assignment, PUBLISHED can become
DRAFT again. Package-row serialization and the locking subscription read remain in place.

Historical subscription snapshots, invoices, payments and trial eligibility remain intact.
Physical deletion still requires a never-assigned draft. No migration or new endpoint is required.
The [client handoff](../../apps/api/docs/api/subscriptions-web-handoff.md#package-contract-and-lifecycle)
and [source plan](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md#package-lifecycle-extension--2026-11-23)
describe the updated rule. Source: the user's report in this conversation.

Verification, 2026-11-24: MySQL-enabled subscription selection passed 332 tests, including 114
MySQL tests, with no failures/errors/skips; bootJar and whitespace checks passed. Coverage includes
unpublish/edit after immediate cancellation, preserved invoices/deletion protection, a remaining
assignment blocking unpublish, and scheduled paid cancellation blocking until completion.

Follow-up: the user explicitly required unpublish/edit while assignments remain, resolved by SUB-021.

### SUB-018 Package trials

**Source/decision date:** 2026-11-23. **Origin:** Explicit user request and answers: trial configuration
contains days, AI minutes and operator minutes; exhaustion blocks only that resource; early payment
starts a fresh paid term immediately; one trial per tenant across all packages. **Status:** Implemented
and verified. The user subsequently authorized commit/push delivery; deployment remains separate. This extends SUB-002/004/008/011/016 while preserving their non-trial behavior.

Add optional package `trial: {days, aiMinutes, operatorMinutes}` to draft/full package DTOs and
snapshot it with the assigned commercial terms. Null/omitted means no trial; a full draft PATCH
can clear it. Days are positive integers and are consecutive 24-hour UTC days from assignment.
Minute pools are nonnegative, at least one positive, and positive pools require their package
calling feature. Those validation details and UTC duration semantics are technical choices.

A first eligible assignment enables enforcement, starts TRIAL immediately, grants the separate
trial minute pools, and creates the normal OPEN invoice for the selected paid term. Keep existing
due-on-issue behavior and do not fabricate payment. Tenant-row serialization plus a locking
historical-trial lookup prevents repeated trials. When already used, a later otherwise valid
assignment follows the regular PENDING flow; this fallback is a technical choice, not a second
trial or a new replacement route. [SUB-019](#sub-019-tenant-subscription-cancellation) later adds cancellation without resetting this trial history. Existing outstanding-invoice restrictions remain.

One resource's exhaustion blocks only that call type. Live usage counts against its pool. When
both are exhausted, all calls block; the subscription remains TRIAL until its date deadline or
payment, so the active flag alone does not promise calling credit. At the deadline, admission
rejects even without worker refresh or any usage. Refresh moves it to PENDING against the same
invoice, records trial completion at the deadline, and grants no paid term. Trial capacities and
non-calling features use the package snapshot; trial periods have no monthly reset or paid grace.
Existing admin enforcement recovery remains an explicit override, not a tenant trial extension.

At payment, including early payment, close the trial and start ACTIVE paid access with its full
allowances and term anchored at settlement. Trial and paid periods are distinct, even if payment
and assignment have the same timestamp. Replays and expiry/payment races must not double-grant.
Calls retain admission-period attribution for late reports. As a technical continuity choice,
already connected trial calls may finish after payment while their usage remains in the trial
ledger; queued trial cutoffs cancel before dispatch on paid activation. Already dispatched provider
termination cannot be recalled. New calls use the paid pool. Top-up quote/order remains paid-ACTIVE
only; there is no trial top-up operation.

The existing leased usage/cutoff workers enforce zero-grace trial minute/day limits. Provider
termination is asynchronous and may overrun deadlines, but new admission is blocked immediately.
The lifecycle worker includes TRIAL so unused trials also expire. Current/admin read models expose
TRIAL, actual trial dates/grants, and effective allowances; `/current.active` is true for unexpired
TRIAL and ACTIVE. Purchased paid terms stay separate from the effective trial minute allowance.
JWT/login remains unchanged. Archive/unpublish/history rules apply to trial-configured packages.

Migration `038-subscription-trials.xml` adds nullable trial timestamps and a PAID-default period
phase; existing records/balances are retained. Replace the original period uniqueness index with
(subscription_id, starts_at, phase), preserving a supporting FK index throughout. Paid activation
at the assignment instant can therefore create a separate paid period. Trial use requires all
backend pods to understand TRIAL; enable/assign only after a coordinated upgrade. Rollout/rollback,
provider timing verification and client integration are separate from local implementation.

Sources: this conversation and the three explicit user answers; [consolidated trial contract](../../apps/api/docs/api/subscriptions-web-handoff.md#trial-packages--2026-11-23);
[source plan](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md#trial-extension--2026-11-23);
[rollout notes](../../apps/api/docs/deployment-gke.md#subscription-trial-rollout).

Verification, 2026-11-23: MySQL-enabled subscription/JWT selection passed 259 tests, including
73 MySQL tests, zero failures/errors/skips. The full MySQL-enabled API suite and bootJar then passed:
1108 passed, zero failures/errors, one intentionally skipped opt-in Vapi live preview test
(`VapiVoiceCatalogLiveTest.actualAdapterCanListValidateAndRetrieveAnAvailablePreviewWithoutProviderMutation`),
including 189 MySQL tests. The full run used a temporary Gradle init script setting the Test heap
to 2 GB (no build configuration change): `./gradlew --init-script
/tmp/nexcall-subscription-trial-test-heap.gradle test bootJar`, with the local MySQL test URL/user.

Trial coverage includes immediate activation with an unpaid invoice, independent pools and
zero-grace minute boundaries, unused day expiry, early payment at the exact assignment timestamp,
late payment, idempotency, once-per-tenant history across packages, current-tenant projection,
day-expiry cutoff queuing/cancellation after payment, late trial usage isolated from paid grants,
populated paid-data migration and concurrent expiry/settlement on MySQL. Diff whitespace checks
passed. No production provider cutoff timing, frontend delivery, commit, push or deployment was
performed in this trial implementation task.

UI handoff expansion, 2026-11-23: at the user's request the same consolidated handoff now includes
an explicit [create → assign → invoice → settlement → current-access walkthrough](../../apps/api/docs/api/subscriptions-web-handoff.md#ui-walkthrough-create-package-assign-tenant-retrieve-invoices-and-settle-payment),
with complete trial/no-trial request examples, the once-per-tenant exception, all per-tenant invoice/
payment retrieval, settlement/retry rules and UI acceptance checks. It documents that there is no
global invoice-list endpoint and that tenant `/current` excludes financial records; no additional
API or permission decision was introduced by this documentation-only follow-up.

Client handoff review, 2026-11-23: at the user's explicit request, the consolidated handoff was
reviewed against the current controllers, DTOs, package validation, billing and access behavior and
reorganized for web/mobile integration only. Removed database/migration details, worker/locking
internals, test counts/commands, rollout procedures, delivery history, seed-offer commentary and
duplicate flow descriptions. Backend evidence remains in this decision record, the implementation
plan and deployment notes. Retained client-visible contracts, role boundaries, nullable dates,
trial eligibility/state semantics, per-tenant invoice retrieval, idempotency, usage units and API
errors; the target-environment availability note is the only deployment-related client guidance.
This documentation review changes no API behavior or permissions.

### Trial delivery — 2026-11-23

At the user's explicit follow-up request, trial implementation SUB-018, migration 038, the reviewed
web/mobile handoff, backend documentation and API credential-file ignore rule were committed on
API `master` as `33c3e3b` (`feat(subscriptions): add tenant trials and client integration handoff`).
The parent `main` delivery records this decision update, indexes SUB-018 and advances only the API
submodule pointer to that commit, with the parent credential-file ignore rule. No branch was created.
The unrelated web checkout and all credential values remain outside this delivery.

Verification is the full API run recorded above: 1108 tests passed, including 189 MySQL tests,
zero failures/errors and one opt-in Vapi live preview skipped; bootJar succeeded. No source or
resource code changed after that run; subsequent edits were documentation and ignore rules.
Push delivery is not confirmation of CI success, deployment or client integration.

### SUB-019 Tenant subscription cancellation

**Source/decision date:** 2026-11-24. **Origin:** Explicit user requirement to remove a tenant's
package at the end of its current paid/trial period, or immediately when no period is active while
cancelling the unpaid invoice; new assignment then follows the normal flow. Explicit clarification:
base payment is blocked while trial cancellation is scheduled and cannot withdraw cancellation.
**Status:** Implemented and verified; source delivery authorized by the user follow-up. Deployment remains separate.
This implements the limited cancellation previously deferred under SUB-014; refunds, reversal and
general replacement remain outside scope.

Add platform-admin POST `/api/subscriptions/customers/{customerId}/subscription/unassign` with the
current `subscriptionId` and an idempotency key. Return the current billing view. A current paid
term schedules cancellation at billingPeriodEnd (not an intermediate monthly allowance boundary),
a current trial at trial.endsAt. Retain ACTIVE/TRIAL access subject to existing quota/capacity limits
until then. With no current paid/trial period, including PENDING/EXPIRED, cancel immediately.

Use the tenant lock to serialize cancellation, payment, lifecycle refresh and assignment. Persist
requested/effective/completed timestamps and the server-derived requesting administrator. Existing
scheduled dates cannot be moved by retries. Payload fingerprints and explicit subscription identity
prevent stale cancellation retries from cancelling a new package. Wrong-tenant references return
the existing not-found response; tenant users cannot mutate subscriptions.

At completion mark the historical subscription CANCELLED, clear its current assignment, preserve
commercial/usage/payment history, and void its wholly unpaid OPEN base/renewal/top-up invoices.
Paid invoices/payments remain untouched; there is no refund. As a financial safety choice, an OPEN
invoice with recorded successful partial payment rejects cancellation with existing invalidPayment
rather than writing off money; broader partial-payment resolution remains deferred. Cancellation
completion enables enforcement, so absence of a package does not become unrestricted observation
access. Explicit later admin recovery remains available under the existing control.

Future renewal invoices and paid terms are suppressed for the cancelled subscription. Base payment
on a scheduled trial returns cancellationPending; valid paid-term top-up settlement can continue
until its original allowance period ends. Replaying an already settled invoice creates no new term.
The normal assignment entry point first completes due cancellation, allowing a fresh package in
that same request; trial eligibility history remains once per tenant across all packages.

Current/admin projections expose nullable cancellation dates while scheduled. At completion their
subscription becomes null, with financial history still in admin billing. Tenant summaries can
report CANCELLED before reconciliation and NONE after detachment. Admission denies new calls at
the effective deadline without waiting for background maintenance; already connected calls retain
existing limits and usage attribution. Existing trial day/minute cutoffs remain in force.

Migration 039 adds nullable cancellation/audit timestamps and actor identity without rewriting
existing financial records. Foreground reads/assignment/settlement and the existing lifecycle worker
complete due cancellations before renewal logic. A coordinated trial/cancellation-aware backend
upgrade is needed before this feature is used; deployment is separate from local verification.

Sources: this conversation and explicit trial-payment clarification;
[client flow](../../apps/api/docs/api/subscriptions-web-handoff.md#cancel-or-remove-a-tenant-subscription);
[source plan](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md#cancellation-extension--2026-11-24).

Verification, 2026-11-24: MySQL-enabled subscription/JWT selection passed all 337 tests, including
113 MySQL tests, zero failures/errors/skips. The full API suite and bootJar then passed 1186 tests,
including 229 MySQL tests, with zero failures/errors and one opt-in Vapi live preview test skipped.
The full run used a temporary 2 GB Test heap through `/tmp/nexcall-subscription-cancellation-test-heap.gradle`
without changing repository build configuration. Coverage includes scheduled full-term/trial removal,
blocked trial payment, immediate pending/expired removal, invoice/payment preservation, partial-payment
protection, scoped/idempotent stale retries, first-request assignment after deadline, Spring-managed
worker completion, populated-data migration 039 and MySQL cancellation/payment/assignment races.
Client JSON examples and cancellation DTO fields were checked against source; whitespace checks
passed. Provider timing, client implementation and deployment remain separate. Source delivery was subsequently authorized by the user; see the delivery record below.

Endpoint naming clarification, 2026-11-24: the user explicitly requested `subscription/unassign`
instead of the initially proposed cancel route. The controller, client handoff and request tests now
use POST `/api/subscriptions/customers/{customerId}/subscription/unassign` only. Request/response
DTOs, cancellation metadata, payment safeguards and effective-date behavior are unchanged.

Verification after the route rename: the focused MVC authorization/request test
`SubscriptionCancellationTest.shouldExposeAdminUnassignEndpointAndDenyTenantMutation` and bootJar
passed. No broader behavioral changes were made and the full suite was not rerun for this rename.

### Unassignment delivery — 2026-11-24

At the user's explicit request, SUB-019 was committed on API `master` as `8946260`
(`feat(subscriptions): support end-of-period tenant unassignment`). It includes migration 039,
POST `/api/subscriptions/customers/{customerId}/subscription/unassign`, cancellation/payment
safeguards, current/admin response metadata, tests and the updated client/backend handoffs.
The parent `main` delivery records SUB-019 and advances the API submodule to this commit.
No new branch was created; the unrelated web checkout and credential values remain excluded.

Evidence remains the full 1186-passed API suite (229 MySQL tests, one opt-in live preview skipped)
and bootJar, followed by the focused renamed-endpoint authorization/request test and bootJar.
Only documentation changed after that final endpoint check. A source push is not confirmation
of CI success, production migration/deployment or completed client integration.

### SUB-021 Edit catalogue packages independently of assigned subscriptions

**Source/decision date:** 2026-11-24. **Origin:** Explicit user requirement to unpublish and
modify packages even while tenants have trial or paid access, retaining each tenant's own copy
of in-progress terms. **Status:** Superseded by
[SUB-022](#sub-022-unpublish-after-all-tenants-request-unassignment) before delivery.
The intermediate implementation was locally verified as recorded below.

Remove the assignment-reference guard from PUBLISHED-to-DRAFT unpublish. This supersedes
SUB-017's never-assigned restriction and SUB-020's intermediate completed-unassignment rule.
Admins can unpublish, edit a DRAFT and republish the same package identity regardless of current
or historical assignments. Repeated unpublish and unpublish of ARCHIVED packages still reject.
Physical deletion still requires a never-assigned DRAFT so historical references remain intact.

Each assignment already stores its own complete terms snapshot. Edits to the catalogue do not
change existing subscription features, prices, allowances, top-up policies, trials, period dates,
invoices, scheduled unassignment or renewal terms. Settling an existing invoice and renewing an
existing subscription use that subscription's snapshot, even if the catalogue price ID has been
removed. There is no automatic migration of existing tenants to revised terms.

DRAFT packages reject new assignments; after republication, new assignments receive the revised
snapshot. Package-row locking continues to serialize assignment with lifecycle edits, so a
concurrent assignment either captures the published version before withdrawal or is rejected
after withdrawal. No migration or endpoint change is required.

The [client contract](../../apps/api/docs/api/subscriptions-web-handoff.md#package-contract-and-lifecycle)
and [implementation plan](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md#package-lifecycle-extension--2026-11-23)
are updated. Clients must not disable unpublish based on assignment counts or tenant status.
Source: this conversation. Existing-subscription repricing remains outside this requirement.

Verification, 2026-11-24: MySQL-enabled `test --tests '*Subscription*' bootJar` passed 356
tests, including 122 MySQL tests, with no failures/errors/skips. Coverage includes all assignment
statuses, pending invoice settlement and renewal using removed catalogue price IDs, original
allowance grants, trial limits and scheduled cancellation surviving edits, new assignments using
the revised configuration, and both concurrent assignment/withdrawal orderings. Diff whitespace
checks passed. The full application suite, UI flows and deployment were not run for this change.

### SUB-022 Unpublish after all tenants request unassignment

**Source/decision date:** 2026-11-24. **Origin:** Explicit user refinement: unpublish/edit is
allowed only for packages no longer assigned, even if assigned previously; unassigned tenants
retain their own terms until the period finishes. **Status:** Implemented and verified locally.
This supersedes SUB-021's unrestricted unpublish and SUB-020's wait-until-access-expires restriction.

A PUBLISHED package can return to DRAFT only if no subscription referencing it remains both
non-CANCELLED and without an unassignment request. Any PENDING/TRIAL/ACTIVE/EXPIRED assignment
without such a request blocks with 409 `termsLocked`. Never-assigned packages remain eligible.
Completed cancellations and scheduled unassignment do not block. Every tenant must satisfy this
rule, so removing one assignment does not unblock a package still assigned to another tenant.

Scheduling unassignment still returns the subscription and cancellation metadata through billing
and current-subscription reads until the existing deadline. Its persisted terms and paid/trial access
remain in force until then, and renewal stays suppressed. This does not clear the current pointer
early or allow that tenant to receive another base subscription before completion. The unpublish
guard interprets the request as removal for catalogue eligibility, independently of remaining access.

After unpublish, admins can edit the DRAFT and republish; future assignments capture revised terms.
Current scheduled subscriptions keep original allowances, trial limits and cancellation deadlines.
Historical records remain intact and physical deletion still requires a never-assigned DRAFT.
The package-row lock and current locking reference read preserve concurrent assignment safety.
No migration, new endpoint or billing DTO change is required.

The [client handoff](../../apps/api/docs/api/subscriptions-web-handoff.md#package-contract-and-lifecycle)
and [source plan](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md#package-lifecycle-extension--2026-11-23)
are updated. Source: this conversation. Deployment and client implementation remain separate.

Verification, 2026-11-24: MySQL-enabled subscription tests and bootJar passed: 336 tests,
including 116 MySQL tests, no failures/errors/skips. Coverage includes blocking remaining
PENDING/TRIAL/ACTIVE/EXPIRED assignments, completed unassignment with retained history, catalogue
edits during scheduled paid/trial removal preserving original terms and deadlines, another
tenant blocking until it also requests removal, and concurrent assignment/withdrawal ordering.
Whitespace checks passed. Full application tests and deployment were not performed for this refinement.

### Subscription catalogue correction delivery — 2026-11-24

At the user's explicit request, the final SUB-022 behavior and consolidated client handoff were
committed and pushed to API `master` as `023e7a5`
(`fix(subscriptions): allow unpublish after all tenants request unassignment`). The parent `main`
delivery updates the canonical decision records and API submodule pointer to this commit.
Intermediate SUB-020/021 policies were superseded before delivery and are retained only as decision history.
Verification is the 336-passed subscription suite above (116 MySQL tests, no skips) and successful
bootJar; only documentation changed after that run. The unrelated web checkout remains excluded.
Source delivery does not confirm CI success, deployment or completed client integration.

### SUB-023 Agent-started calls use AI minutes through caller hang-up

Superseded for newly admitted calls by [SUB-024](#sub-024-split-transferred-calls-between-ai-and-operator-allowances).

**Decision and source** — Explicit user instruction, 2026-11-25, following the live recording and
duration check on inbound interaction 28. A call that starts with an agent uses the AI pool for its
entire connected duration, including human handover, even when the tenant has zero operator minutes.

**Status** — Implemented in the local checkout for newly admitted inbound and outbound agent calls.
The original Twilio caller leg supplies the terminal duration. The session stays in its admission
period and resource pool. A verified parent completion settles the total once. When carrier identity
or final duration is missing, reconciliation can retrieve it. A known undialed cancellation settles
at zero. The separate operator leg and recording are unnecessary for this calculation.
Existing sessions retain their admission-time policy. No historical debit is authorized.

This supersedes the former AI-end boundary for agent calls in SUB-005 and resolves the open human-leg
billing question in AGENT-011. CALL-020 describes the interaction display. Live validation of the
new billing behavior remains pending.
[Client contract](../../apps/api/docs/api/subscriptions-web-handoff.md#agent-calls-and-human-transfers).

Within SUB-023, a signed completion callback supplies a provisional duration. The worker checks
recent provisionally settled caller sessions within the two-hour recovery window. A dated Twilio
Call resource supersedes callback evidence; a newer verified version can correct usage up or down.
Apply the difference in the original allowance period. Equal or older verified versions and
subsequent callback replays cannot undo the correction. An agent report can bind a previously
unknown caller SID while billing is awaiting usage and make the session due for reconciliation.

API master commit `f7e148d` includes both corrections. The full MySQL-enabled build passed
1,388 tests with one optional live test skipped. Deployment and live balance verification are separate.

### SUB-024 Split transferred calls between AI and operator allowances

**2026-11-25 · Explicit user instruction · Implemented in checkout, pending deployment.** This replaces [SUB-023](#sub-023-agent-started-calls-use-ai-minutes-through-caller-hang-up) for the applicable calls. Inbound and outbound AI conversations consume AI seconds. Time actually connected to a human consumes operator seconds and can take the operator balance below zero. An unanswered or declined transfer has zero human conversation time. The displayed total duration still follows the original caller through hang-up under CALL-020 and CALL-025.

Source: the user's follow-up after campaign 73. That outbound call has a connected Vapi transfer artifact with a timestamped successful screening result. Inbound NexCall records spoken acceptance and receives Twilio bridge confirmation. The user confirmed that ringing and screening consume AI seconds. Transfers on trials may overdraw the operator balance until the trial deadline. The split applies only to calls admitted under the new policy. Calls already in flight and settled historical usage retain SUB-023. Implementation uses the original session, subscription period and concurrent slot. Late bridge or carrier evidence adjusts the two ledger entries by their differences. See the [client contract](../../apps/api/docs/api/subscriptions-web-handoff.md#agent-calls-and-human-transfers).

Delivery, 2026-11-26: API master `3c98aab` and web master `7975134` include SUB-024 and CALL-025. The local API build passed with 1,396 tests passed and one skipped. The web build and 69 subscription tests passed. Two unrelated web tests (cold-calling table columns and the breadcrumb translation pipe) also failed on the unmodified web revision. Live transfer and billing verification is pending.
