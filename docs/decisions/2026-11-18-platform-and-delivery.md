# Platform and delivery decisions

Consolidated on 2026-11-18 (Europe/Bucharest). Status meanings and provenance rules are in the
[decision index](README.md).

### PLAT-001 Preserve independent application repositories

**Status:** Accepted · Repository instruction; implemented structure.

The parent tracks Angular web, Spring API and Flutter mobile as independent Git submodules.
Keep each application's dependency manager, build tools, lockfiles and history. Do not introduce
cross-application source imports or incidental toolchain upgrades. Git actions and deployments
require the user's authorization. This preserves independent verification and delivery.

Sources: [root guidance](../../AGENTS.md), [repository ownership](../monorepo.md#repository-ownership).

### PLAT-002 Run one modular monolith with local module boundaries

**Source dates:** 2026-07-07 and 2026-07-23. **Status:** Implemented · Architecture choice.

Run the API as one root Gradle application with domain modules in packages and local
`*ModuleDependencies` facades. Preserve external endpoint paths and separate persistence units.
The original plan's Java 11, separate Gradle library modules, port/adapters and dual runtime
profiles are historical, superseded by the current root application and Java 25 baseline.
Framework versions remain owned by the actual build, not old plans.

Sources: [original migration](../../apps/api/docs/superpowers/plans/2026-07-07-modular-monolith-migration.md),
[API architecture](../../apps/api/README.md#module-boundaries), [build](../../apps/api/build.gradle),
[settings](../../apps/api/settings.gradle).

### PLAT-003 Deploy applications independently on GKE

**Source dates:** 2026-07-23 and 2026-07-26. **Status:** Implemented configuration · Technical choice.

Build the API as one JVM image and the web as static Angular assets served by Nginx with SPA
fallback. Use each application's Helm chart and the root deployment workflows where applicable.
Keep secrets server-side. The July deployment plan's temporary Hibernate `update` policy is
superseded for the customer/users schemas by Liquibase and validation. Do not treat old branch,
host or framework values in a plan as current configuration or proof of a live deployment.

Sources: [API deployment plan](../../apps/api/docs/superpowers/plans/2026-07-23-github-actions-kubernetes-deployment.md),
[web deployment plan](../../apps/web/docs/superpowers/plans/2026-07-26-nexcall-ui-gke-deployment.md),
[web monolith compatibility](../../apps/web/docs/superpowers/plans/2026-07-26-nexcall-api-monolith-compatibility.md),
[API deployment guide](../../apps/api/docs/deployment-gke.md), [web deployment guide](../../apps/web/docs/deployment-gke.md).

### PLAT-004 Optional Firebase and browser-specific Twilio tokens

**Source date:** 2026-07-27. **Status:** Implemented · Technical choice.

Missing Firebase credentials disable push delivery while preserving startup and device record
operations. Browser Voice SDK clients request `WEB` Twilio tokens containing the TwiML App SID
without mobile push credentials. Keep Android/iOS token support separate and issue tokens only
on the server. This removes mobile-only credential requirements from browser calling.

Sources: [Firebase plan](../../apps/api/docs/superpowers/plans/2026-07-27-optional-firebase-credentials.md),
[browser token plan](../../apps/web/docs/superpowers/plans/2026-07-27-web-twilio-access-token.md),
[Firebase configuration](../../apps/api/src/main/java/com/nexcall/monolith/modules/notifications/config/FirebaseDisabledConfiguration.java),
[Twilio service](../../apps/api/src/main/java/com/nexcall/monolith/modules/twilio/service/TwilioService.java).

### PLAT-005 Replace Azure storage with Google Cloud Storage

**Source date:** 2026-11-13. **Status:** Implemented · Migration choice.

Use private GCS buckets for profile photos and CRM attachments with workload identity and IAM
signing. Photos use 30-minute signed GET URLs. CRM writes are create-only, downloads stay behind
tenant authorization, and orphan cleanup remains durable. Preserve the client response fields.
This replaces the Azure storage retained in historical [CRM-022](2026-11-05-crm-backend.md#crm-022--api-simplification-backlog).
Production bucket/IAM setup and provider checks remain deployment work. Historical cleanup of
stale object references requires a separate decision.

Sources: [storage migration](../../apps/api/docs/superpowers/plans/2026-11-13-azure-to-google-cloud-storage.md),
[storage implementation](../../apps/api/src/main/java/com/nexcall/monolith/shared/dto/services/CloudStorageService.java),
[CRM storage](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/crm/CrmAttachmentStorage.java).

### PLAT-006 Archive orphaned tenant admins and preserve identity history

**Source date:** 2026-11-13. **Status:** Implemented · Lifecycle choice; rollout is gated.

Deleting a tenant revokes its admin memberships and queues user archival only when the admin has
no membership in another live tenant, including suspended memberships. Retain old IDs and
historical email snapshots; permit email reuse after archival commits. Authorize HTTP, refresh
and STOMP by signed user ID, not reusable email. Coordinate the two databases through durable
revocation work. Drain older replicas before enabling archival/email reuse and retain an
ID-aware rollback image. Cleanup of tenants deleted before rollout remains a separate decision.

Sources: [lifecycle plan](../../apps/api/docs/superpowers/plans/2026-11-13-tenant-admin-soft-delete.md),
[revocations](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/TenantAdminRevocations.java),
[lifecycle tests](../../apps/api/src/test/java/com/nexcall/monolith/api/TenantAdminLifecycleTest.java).

### PLAT-007 Preserve business date and diagnostic timestamp contracts

**Status:** Implemented · Existing API contract; original approval date not established here.

Use `dd/MM/yyyy` for business dates and `dd/MM/yyyy HH:mm` for business date-times. Technical
events and billing boundaries retain their ISO timestamp contracts. Provider-specific formats
remain provider-specific. Do not heuristically accept both month-first and day-first dates.

Sources: [API date contract](../../apps/api/README.md#api-date-contract),
[date constants](../../apps/api/src/main/java/com/nexcall/monolith/shared/utils/Constants.java).

### PLAT-008 Domain packages and shared web presentation

**Status:** Implemented · Existing web architecture; original approval date not established here.

Keep the Angular application as the host for routing, authentication and runtime configuration.
Domain packages own their presentation, NgRx state and data access through public package APIs;
reusable controls belong to UX, with shared calling contracts and workflow UI kept separate.
Libraries receive host configuration rather than importing application environments. Preserve
lazy loading, permissions and each package's independent build. Angular 15/TypeScript 4.9 and
the pinned Node/npm configuration remain the current toolchain; a feature extraction is not
permission to upgrade them or introduce unsupported standalone/Signals patterns.

Sources: [web architecture](../../apps/web/docs/architecture.md),
[package structure](../../apps/web/docs/structure.md), [build guide](../../apps/web/docs/build-and-run.md).

### DOC-001 Maintain decisions centrally and send only contract differences to web

**Decision date:** 2026-11-18. **Status:** Accepted · Explicit user instruction; documented here.

Keep current decisions under the parent's `docs/decisions`, reconcile outdated statuses and
maintain the log during future tasks. Preserve origin and historical replacements. Create a
web handoff only when the backend differs from the web proposal; document those differences
instead of repeating unchanged contracts. Plans and handoffs support the decision log rather
than replacing it.

Sources: the user's instructions in this chat, [root guidance](../../AGENTS.md#decision-log),
[subscription contract differences](../../apps/api/docs/api/subscriptions-web-handoff.md).
