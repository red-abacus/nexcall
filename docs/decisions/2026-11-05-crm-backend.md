# NexCall CRM and campaign decisions

The initial decisions were agreed on 2026-11-05 (Europe/Bucharest) and recorded at `2026-11-05T14:24:12Z` (UTC). A product choice is an explicit user decision. A plan default is a technical choice accepted with the implementation plan. Keep superseded entries and link them to their replacements.

## Shared API and Web

| ID | Status and origin | Decision | Why and consequence | Reference |
| --- | --- | --- | --- | --- |
| CRM-001 | Accepted · Product choice | Ship persistent CRM and full campaign prototype parity. | CRM is useful in the live calling workflow. The release includes accounts, contacts, catalogs, updates, linked call history, cold campaigns and scripts, CRM and CSV imports, and bulk removal. | [CRM handoff](../../apps/web/docs/crm-phase-1-backend-handoff.md), [Campaign import contract](../../apps/web/docs/campaign-call-task-imports.md) |
| CRM-002 | Accepted · Product choice | Platform admins select one tenant. Client admins use their membership. Campaign operations use the campaign tenant. | The API principal has no active tenant for platform admins. Lists, writes, relationships, downloads, and import searches must all enforce the resolved tenant. | [Customer service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CustomerService.java) |
| CRM-003 | Accepted · Product choice | Archive accounts and contacts. Unlink an archived account’s contacts. Cancel an archived contact’s queued tasks and let active attempts finish. | This preserves historical attribution while removing archived records from active views and dialing. Every path back to the queue cancels an archived contact’s task. | [CRM handoff](../../apps/web/docs/crm-phase-1-backend-handoff.md), [Call task service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CallTaskService.java) |
| CRM-004 | Accepted · Product choice | Require a leading plus and normalize new CRM, prospect CSV, and cold manual-task phones to international form. | The tenant lacks structured country data for local dialing. Legacy debt-recovery inputs retain their existing contract. | [Customer entity](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/entities/Customer.java) |
| CRM-005 | Accepted · Plan default | Keep legacy campaigns as debt recovery and add immutable cold calling with one tenant-scoped CALL script. | Campaign type determines the manual form, script selection, and export schema. Existing disposition and Twilio setup remain compatible. | [Campaign import contract](../../apps/web/docs/campaign-call-task-imports.md) |

### CRM-017 · Limit linked history to verified outbound attempts

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T14:55:51Z` UTC
- **Status** Accepted · Implementation decision
- **Decision** Record CRM dispositions for outbound attempts with a verified attempt key. Keep provider-scoped inbound routing and let callbacks release inbound tasks.
- **Rationale and consequence** The inbound notification arrives separately from the SDK invite and carries no trusted leg correlation. The API requires a verified outbound attempt key to record a linked disposition.
- **Reference** [Operator call effects](../../apps/web/modules/operator/src/lib/store/twilio.effects.ts), [Disposition service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CallDetailsService.java)

### CRM-018 · Keep old clients eligible for compatible tasks

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T14:55:51Z` UTC
- **Status** Accepted · Implementation decision
- **Decision** The API accepts `crmCapable=true` from compatible clients on next-task reads. Requests without it see legacy unlinked debt-recovery tasks.
- **Rationale and consequence** Older clients lack the attempt key and cold script support. Filtering the queue avoids presenting a task to earlier clients that cannot dial it. The restored Angular implementation sends the capability flag.
- **Reference** [Operator next-task endpoint](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/controllers/OperatorCallTaskController.java)

## API

| ID | Status and origin | Decision | Why and consequence | Reference |
| --- | --- | --- | --- | --- |
| CRM-006 | Accepted · Product choice | Publish CRM history only for saved operator dispositions, once per linked call attempt. | A task can return to the queue after a disposition. The persisted attempt identifies the outcome, dialed number, and operator even when the same task is retried. | [Call disposition service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CallDetailsService.java) |
| CRM-007 | Accepted · Product choice | Save the disposition and outbox event together. Retry delivery to CRM independently. | A CRM write failure should not cause a call outcome save to fail. History is eventually consistent and the oldest pending event is monitored. | [Integration guidelines](../../apps/api/docs/guidelines/integrations-background.md#workflow-ordering-and-database-coordination) |
| CRM-008 | Accepted · Plan default | Adopt additive Liquibase migrations in the customer persistence unit, with the bundled CAEN Rev. 3 and NAICS US 2022 datasets. | This places CRM and calling writes in one transaction boundary and preserves classification versions and string codes. | [Customer persistence config](../../apps/api/src/main/java/com/nexcall/monolith/config/jpa/CustomerJpaConfig.java), [Catalog data](../../apps/web/modules/settings/assets/industry-classifiers/catalog.json) |
| CRM-009 | Superseded by [CRM-022](#crm-022--api-simplification-backlog) · Plan default | Validate and scan attachments before private Azure Blob upload, then commit metadata with the immutable update. | Users receive a successful update once its attachments are available. Failed uploads and interrupted writes need cleanup. | [CRM attachment contract](../../apps/web/docs/crm-phase-1-backend-handoff.md#update-and-attachment-endpoints) |
| CRM-010 | Accepted · Plan default | Bind preview tokens and idempotency keys to their authorized scope. Recheck selection at import and serialize task changes with outbound activation. | Exact retries return the same result, archived contacts stay out of the queue, and a bulk removal cannot race a call start. | [Campaign import contract](../../apps/web/docs/campaign-call-task-imports.md) |

### CRM-013 · Adopt the customer schema baseline

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T14:37:06Z` UTC
- **Status** Accepted · Implementation decision
- **Decision** Use the MySQL customer schema from API `58ffedf` as the Liquibase bootstrap baseline. Existing customer schemas adopt that baseline and run the additive changesets.
- **Rationale and consequence** Customer tables previously came from Hibernate schema updates. This supports a clean install and preserves existing rows. Hibernate validates the customer schema after migrations. Other database schemas keep their existing lifecycle.
- **Reference** [Customer baseline](../../apps/api/src/main/resources/db/changelog/000-customers-baseline.xml), [Customer persistence configuration](../../apps/api/src/main/java/com/nexcall/monolith/config/jpa/CustomerJpaConfig.java)

### CRM-014 · Persist campaign request receipts and use single-use CSV previews

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T14:46:25Z` UTC
- **Status** Accepted · Implementation decision
- **Decision** A successful import consumes its preview token. An exact retry first looks up the stored response. Bulk deletion fingerprints the sorted unique task IDs. A nullable generated index enforces live contact uniqueness per campaign.
- **Rationale and consequence** This preserves safe retries after an ambiguous response and lets admins re-import a removed contact. The database enforces uniqueness even if concurrent requests reach the service.
- **Reference** [Campaign task operations](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CampaignTaskOperations.java), [Campaign migration](../../apps/api/src/main/resources/db/changelog/002-campaign.xml)

### CRM-015 · Bound the wait for provider duration and make upload cleanup durable

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T14:46:25Z` UTC
- **Status** Accepted · Implementation decision
- **Decision** Give the outbox one minute to observe the provider duration. After that, publish with unknown duration if necessary. Reserve orphan cleanup before external upload with a 15-minute lease.
- **Rationale and consequence** A late provider callback cannot stall the history indefinitely. Interrupted attachment requests can be retried while a worker cleans up unreferenced blobs.
- **Reference** [Disposition outbox](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/crm/CrmCallUpdateOutbox.java), [Contact history persistence](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/crm/CrmContactHistory.java)

### CRM-016 · Store linked attempts with one managed persistence write

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T14:46:25Z` UTC
- **Status** Accepted · Implementation decision
- **Decision** Lock the task and persist the new call attempt once, with the task update flushed by the same transaction. Bind the linked browser attempt to signed provider identity.
- **Rationale and consequence** The persistence context tracks the locked task. A second explicit save of the same transient child caused duplicate attempt inserts during the packaged smoke test.
- **Reference** [Call details service](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CallDetailsService.java), [Webhook signature verification](../../apps/api/src/main/java/com/nexcall/monolith/modules/twilio/service/TwilioWebhookSignature.java)

### CRM-019 · Scope conference reads and treat imported account IDs as snapshots

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T14:55:51Z` UTC
- **Status** Accepted · Implementation decision
- **Decision** Validate protected call and conference reads against campaign access and ownership. Keep imported account attribution as a snapshot while enforcing the contact link and tenant scope on import.
- **Rationale and consequence** Existing conference routes now handle CRM-linked data. A separate account snapshot also avoids reversing lock order when an account is archived concurrently with import.
- **Reference** [Conference access policy](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/services/CallTaskService.java), [Campaign migration](../../apps/api/src/main/resources/db/changelog/002-campaign.xml)

### CRM-022 · API simplification backlog

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T16:57:51Z` UTC
- **Status** Implemented · User-approved simplification
- **Decision** Simplify the backend that serves the existing Angular CRM and campaign endpoints. Preserve the web contract, tenant authorization, and reliable retries.
- **Rationale and consequence** Spring binding, repositories, and transaction events can handle much of the plumbing. The list below records the changes for a separate API implementation pass.
- **Reference** [API architecture guidelines](../../apps/api/docs/guidelines/architecture-java-style.md#responsibilities), [DTO guidelines](../../apps/api/docs/guidelines/dto-validation-mapping.md), [Integration guidelines](../../apps/api/docs/guidelines/integrations-background.md#workflow-ordering-and-database-coordination)

1. **Remove ClamAV.** Keep multipart uploads, file size and type checks, private Azure Blob storage, tenant-scoped downloads, idempotency, and cleanup of interrupted uploads. Remove the scanner client, Helm settings, tests, and rollout instructions. Basic file validation cannot detect malware in an otherwise valid file.
2. **Replace `CrmInput` with typed account and contact request DTOs.** Use Spring and Bean Validation for binding and field checks. Preserve omitted, explicit null, and supplied values on PATCH, and validate the merged record in the service.
3. **Move persistence into repositories and batch CRM imports.** Use Spring Data for ordinary lookups and a shared customer `JdbcClient` in focused repositories for MySQL-specific locking and projections. Batch contact locks and reads while preserving tenant scope and lock order.
4. **Use typed campaign and list requests.** Bind import and bulk-delete requests through Spring and share a typed contact-rule model between search and import. Reuse `QueryResult<T>` for ordinary lists. Preserve the existing query parameters, response fields, and translation keys. Cover malformed input and PATCH presence in MVC tests.
5. **Publish script refresh notifications after commit.** Use a Spring after-commit transaction listener for operator refreshes. The disposition outbox continues to deliver CRM call history independently of saving the outcome.
6. **Group the remaining CRM settings.** Bind related settings with validated `@ConfigurationProperties` and preserve the deployment environment variable names.

### CRM-023 · Preserve import receipts through typed binding

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T17:19:13Z` UTC
- **Status** Implemented · Technical default
- **Decision** Bind CRM contact rule groups to a typed model. Retain their original JSON representation for campaign request fingerprints.
- **Rationale and consequence** Existing receipts hash the nested rule object's field order and values. Retaining that representation lets clients replay saved all-match imports across the API rollout.
- **Reference** [Contact rule binding](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/crm/CrmContactRules.java), [Import receipt regression](../../apps/api/src/test/java/com/nexcall/monolith/modules/customers/services/CampaignTaskOperationsTest.java)

### CRM-024 · Batch contact locks and recheck selection

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T17:19:13Z` UTC
- **Status** Implemented · Technical default
- **Decision** Lock the selected active contact IDs in a tenant-scoped, ascending ID scan. Fetch the callable snapshots and ordered phones in batches, and apply the picker predicates again before taking the campaign lock.
- **Rationale and consequence** Large imports avoid per-contact database round trips. Edits or archives committed before the contact locks are acquired can remove a contact from the import selection. The contact, campaign, task, and attempt lock order still applies.
- **Reference** [Contact repository](../../apps/api/src/main/java/com/nexcall/monolith/modules/customers/crm/CrmContactRepository.java), [MySQL integration tests](../../apps/api/src/test/java/com/nexcall/monolith/modules/customers/crm/CrmMySqlIntegrationTest.java)

## Web

| ID | Status and origin | Decision | Why and consequence | Reference |
| --- | --- | --- | --- | --- |
| CRM-011 | Restored by [CRM-021](#crm-021--restore-the-angular-changes) · Plan default | Use live feature flags, server paging, tenant-aware deep links, and exact-retry keys. | Mock switches control interceptors. The live interface must handle production-size collections and preserve tenant scope and drafts across navigation and retries. | [CRM routing](../../apps/web/modules/crm/src/lib/crm.routing.ts), [Campaign import service](../../apps/web/modules/calls/src/lib/data/services/campaign-import.service.ts) |

### CRM-020 · Limit delivery to the API

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T15:48:08Z` UTC
- **Status** Superseded by [CRM-021](#crm-021--restore-the-angular-changes) · Prior scope interpretation
- **Decision** Deliver backend CRM and campaign support in the API submodule. Restore the Angular submodule to its baseline and track a live web cutover separately.
- **Rationale and consequence** The existing web app is the prototype contract for the backend. Its CRM and campaign controls still use the existing mock gates. The API can be verified independently using authenticated test clients. This supersedes the Angular implementation in CRM-011 and narrows the current delivery scope of CRM-001.
- **Reference** [CRM prototype routing](../../apps/web/modules/crm/src/lib/crm.routing.ts), [API rollout guide](../../apps/api/docs/deployment-gke.md#crm-rollout)

### CRM-021 · Restore the Angular changes

- **Decision date** 2026-11-05
- **Recorded at** `2026-11-05T16:04:45Z` UTC
- **Status** Accepted · Explicit user authorization
- **Decision** Restore the Angular implementation and retain the API work. Keep the existing live rollout gates.
- **Rationale and consequence** The user clarified that the original question asked for an explanation of the web changes. They then explicitly authorized restoration. The recovered Angular changes match the backup exactly.
- **Reference** [CRM cutover notes](../../apps/web/docs/crm-phase-1-backend-handoff.md#web-cutover-controls), [API rollout guide](../../apps/api/docs/deployment-gke.md#crm-rollout)

## Mobile

| ID | Status and origin | Decision | Why and consequence | Reference |
| --- | --- | --- | --- | --- |
| CRM-012 | Accepted · Product choice | Defer Flutter support for the new CRM and cold-calling functionality. | The mobile client is several versions behind and will be addressed separately. This delivery verifies the API and Angular clients. | [Mobile manifest](../../apps/mobile/pubspec.yaml) |

## Implementation baseline

[API code and data model review](../reviews/2026-11-05-crm-api-review.md)

The reviewed API checkout was `58ffedf`. The parent recorded `5673991` for its API gitlink at the start of implementation.

## Verification handoff

Verified locally at `2026-11-05T15:02:47Z` UTC.

- API `./gradlew test bootJar` passed 190 tests with zero failures or skips. Eight MySQL integration cases covered fresh bootstrap, legacy adoption, tenant scope, task locking, request replay, history delivery, and attachment failure recovery.
- The packaged API started against fresh MySQL. Authenticated smoke requests exercised CRM CRUD, tenant isolation, manual history, cold scripts, both import sources, bulk removal, the legacy task-queue filter, protected conference reads, and two signed synthetic Twilio attempts. Disposition replay and conflict, stale callback isolation, provider duration, and both history deep links passed.
- Angular passed 303 affected Chrome Headless checks, independent package builds, lint, and staging AOT build `66177396b7d9eda9` before the revert. The restored tracked diff and added files matched the saved backup. The deployment Helm chart passed lint with CRM enabled.
- The initial local tests exercised signed synthetic Twilio requests, a loopback ClamAV protocol fixture, and attachment storage failure and recovery. The API and Angular rollout flags default to off.

### CRM-022 verification

Recorded at `2026-11-05T17:19:54Z` UTC.

- API `./gradlew test bootJar --offline` passed 204 tests with zero failures or skips. Nine tests used disposable MySQL 8.4.13 schemas and covered schema adoption and bootstrap, tenant isolation, lock contention, selection rechecks, PATCH persistence, request receipts, and history and attachment recovery. The MVC tests covered malformed payloads, paging, and internal PATCH field protection. The transaction tests verified script refresh on commit and suppression on rollback.
- The CRM-enabled Helm chart passed lint and rendered the 52 MiB NGINX upload limit and private storage settings. Live Azure Blob and Twilio still require the provider-backed checks in the [CRM rollout guide](../../apps/api/docs/deployment-gke.md#crm-rollout).
