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
changed terms require a new package identity. Archiving preserves prior purchases.

Evidence: [package service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionPackageService.java),
[billing service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/subscriptions/SubscriptionBillingService.java).

### SUB-002 Manual invoice settlement

**Status:** Accepted and implemented · Agreed requirement and supplied web contract.

Assignment creates PENDING access and an OPEN invoice. Only external-payment confirmation
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

Support 1/3/12-month terms, RECURRING or ONE_TIME, and MONTHLY or PER_TERM allowances. Monthly
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
admins. It is shared connected time per tenant/resource/admission period, not per call. Block
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
PENDING access blocks new calls. Base settlement enables enforcement again. Platform admins have
an explicit enforcement recovery toggle; disabled enforcement still measures usage. This is the
current implementation, replacing the draft idea that assignment and enforcement activation
would always be separate steps. Pilot rollout and provider checks remain operational work.

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

Automated collection, payment-provider checkout, refunds, general cancellation/replacement,
discounts, customer self-service, tax/gross fiscal invoice models, arbitrary credit editing and
minute reservations are outside this delivery. The current ledger is net operational billing.
Web/mobile integration is separate from the API implementation; a backend record is not proof
that those clients or production provider flows have been delivered.

Sources: [plan deferred extension](../../apps/api/docs/superpowers/plans/2026-11-18-tenant-subscriptions-and-call-limits.md),
[web contract changes](../../apps/api/docs/api/subscriptions-web-handoff.md).
