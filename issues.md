# Issues

## 1. Recording playback for inbound and outbound calls

- **Status:** Open
- **Reported:** 2026-11-18
- **Section:** Inbound and outbound campaigns
- **Inbound campaign ID:** 48
- **Interaction ID:** 2

### Steps to reproduce

1. Open inbound campaign **48**.
2. Find inbound call interaction **2**.
3. Click the action to listen to its call recording.

**Expected:** Recording playback for both inbound and outbound calls should be available when the recording option is enabled in the UI.

**Actual:** The recording request returns HTTP **404** with message `voiceAgent.recordingUnavailable`.

### Error response

Endpoint: `https://api.nexcall.abac.software/api/inbound/agents/48/interactions/2/recording`

```json
{
    "details": null,
    "locale": "en-GB",
    "message": "voiceAgent.recordingUnavailable",
    "path": "https://api.nexcall.abac.software/api/inbound/agents/48/interactions/2/recording",
    "timestamp": "2026-11-18T15:35:47.955056843"
}
```

**Investigation:** Pending. Reported behavior has not yet been independently reproduced; the cause is undetermined.

## 2. Saving the update after an inbound interaction to CRM fails

The failing operation is saving the update produced after an inbound interaction to CRM. For the reported existing phone number, this fails with `pages.crm.errors.invalidName`.

- **Status:** Open
- **Reported:** 2026-11-18
- **Section:** Inbound campaigns → Save to CRM
- **Task ID:** 53
- **Inbound campaign ID:** Not provided for this issue

### Steps to reproduce

1. Open a completed inbound interaction whose phone number already exists in the CRM.
2. Click **Save to CRM** to save the update produced by that interaction.
3. The UI prompts for a name and details.
4. Click **Save**.

**Expected:** When the phone number already belongs to a CRM contact, transfer the update produced by the inbound call to that existing contact. This is a sync operation, not new-contact creation.

### Required lookup and form flow

1. When opening **Save to CRM**, look up CRM contacts using the inbound caller's phone number, scoped to the current tenant. Use consistent phone-number normalization so equivalent number formats match.
2. If a contact exists, fetch its existing CRM details and pre-populate the form. Clearly indicate that saving will add the interaction to that contact's updates.
3. Display already completed contact information as grayed-out, read-only fields. This interaction-saving flow must not overwrite established CRM details. Missing information can remain editable where supported; the exact protected fields and rules for filling missing values must be agreed in the API/UI contract.
4. Save the call interaction as an actual update linked to the matched CRM contact, preserving its existing details. Do not create a duplicate contact or apply new-contact name validation to an existing-contact update.
5. If no contact matches, show the new-contact form and validate the required fields before creating the contact and attaching the interaction.

- Show loading and lookup failure feedback. A failed lookup must not be treated as “no matching contact.” If multiple contacts match, require an explicit resolution rather than silently choosing one.
- The backend must return enough information to distinguish an existing-contact update from new-contact creation and identify the matched contact. Exact lookup and save endpoints/payloads remain to be defined.
- Task-level failures must not be treated as a successful sync merely because the HTTP status is 200.

### Acceptance

- An existing phone number pre-populates the correct contact's CRM details before saving.
- Completed protected fields are visibly read-only, and saving leaves those values unchanged.
- The interaction appears in the existing contact's updates without creating another contact.
- An unknown phone number follows the new-contact flow; lookup errors and ambiguous matches are handled explicitly.

**Actual:** The PATCH request returns HTTP **200**, but its response reports task **53** as failed with `pages.crm.errors.invalidName`. No tasks are reported as saved or duplicate. The exact submitted name/details and the UI behavior after the response have not been provided.

### Response body

```json
{
    "savedTaskIds": [],
    "duplicateTaskIds": [],
    "failedTasks": [
        {
            "taskId": 53,
            "message": "pages.crm.errors.invalidName"
        }
    ]
}
```

**Investigation:** Pending. Check tenant-scoped phone-number lookup and normalization, pre-population of CRM details, protected/read-only field rules, missing-value completion, existing-contact update versus contact creation payloads, transfer of the call update, backend name validation, and handling of task-level failures in a successful HTTP response. Reported behavior has not yet been independently reproduced; the cause is undetermined.

## 3. Backend usage must return used and remaining seconds for all usable allowances

- **Status:** Pushed to API master in `d427409`. Web integration and live verification pending.
- **Reported:** 2026-11-24
- **Section:** Usage
- **Owner:** Backend

**Requirement:** The backend usage response must return both the amount of used seconds and the remaining seconds for every usable allowance.

**Acceptance:** Every usable allowance is identifiable in the usage response and includes both used seconds and remaining seconds, including zero values.

**Implementation:** The subscription usage object now exposes signed AI and operator remaining seconds alongside their used seconds. The API regression suite covers live usage, zero and overrun balances. [Web integration contract](apps/api/docs/api/settings-usage-web-handoff.md).

## 4. Agent setup statuses

- We need better statuses for when the agent is being set up, using the statuses received through the WebSocket.

## 6. Backend Twilio number list needs paging, sorting, and filtering

- **Status:** Pushed to API master in `d427409`. Web integration and live verification pending.
- **Reported:** 2026-11-18
- **Section:** Settings → Twilio Numbers
- **Owner:** Backend, with UI integration after the API contract is available
- **Endpoint:** `GET /api/settings/twilio-numbers`

**Original gap:** Pagination was missing. The previous standalone pagination row marked OK in `verify.md` was incorrect and has been removed.

**Required work:** Support server-side paging, sorting, and filtering for the phone-number list. Define supported query parameters and return page metadata, including the total matching count, so the UI can navigate results correctly. Apply filtering and sorting before paging and preserve tenant scoping and permissions.

**Acceptance:** The list can request different pages and page sizes, sort supported columns consistently, and filter across the full matching inventory rather than only the loaded page. Integrate the UI table with the agreed API contract.

**Implementation:** The settings endpoint supports scoped database paging, sorting and inventory search. Existing unpaged requests keep their array response. The web team owns table integration. [Web integration contract](apps/api/docs/api/settings-usage-web-handoff.md).

## 9. Industry Classifiers needs pagination, sorting, and filtering

- **Status:** Pushed to API master in `d427409`. Web integration and live verification pending.
- **Reported:** 2026-11-19
- **Section:** Settings → Industry Classifiers
- **Owner:** Backend and frontend

**Requirement:** Provide pagination, sorting, and filtering for industry-code lists under both CAEN and NAICS.

**Current context:** The frontend contains local pagination and code/description search over loaded entries. This does not establish complete server-side pagination, sorting, or filtering; verify the live API contract and connect the page to it.

### Required work

- Support page selection, page size, and total matching counts.
- Allow sorting by industry code and description in ascending or descending order, with stable ordering across pages.
- Support filtering/search by code and description within the selected standard, across the complete dataset rather than only the loaded page.
- Define the backend query parameters and response metadata, and wire the table controls to that contract.
- Reset to the first page when the standard, filters, or sorting changes; retain the selected criteria while navigating pages.

**Acceptance:** Users can browse, sort, and filter both standards with accurate totals and consistent results across pages. Empty results and request failures are clearly presented.

**Implementation:** CAEN and NAICS support server-side code/description search, stable sorting and matching totals. The web team owns table integration. [Web integration contract](apps/api/docs/api/settings-usage-web-handoff.md).

## 10. Preset visibility and filtering must respect tenant ownership

- **Status:** Pushed to API master in `d427409`. Web integration and live verification pending.
- **Reported:** 2026-11-19
- **Section:** Settings → Presets
- **Owner:** Backend and frontend

**Requirement:** Platform admins must be able to see presets from all tenants, identify which tenant owns each preset, and filter the list by tenant. Tenant users must see only their own tenant's presets.

### Required work

- Allow platform admins to list presets across tenants, with an all-tenants view and a tenant filter.
- Include tenant ownership in the API response and display the owning tenant clearly for each preset in the admin list.
- Restrict tenant users to presets belonging to their authenticated tenant. Do not rely on a frontend filter for isolation.
- Enforce tenant access on backend list/search and individual preset access, including direct requests with another tenant's preset ID or filter value.
- Apply the same visibility rules to preset selectors elsewhere in the application.

**Acceptance:** An admin can see presets from multiple tenants, distinguish their owners, filter to one tenant, and clear that filter. A tenant user sees only their own tenant's presets and cannot access another tenant's presets by modifying a request or navigating directly to a record.

**Scope note:** Cross-tenant visibility does not by itself define new admin edit/archive permissions. The agreed tenant exception covers the 16 read-only built-ins.

**Implementation:** Platform Settings exposes preset ownership and exact-tenant filtering. Tenant administrators can access their own presets and the 16 built-ins. Ownerless custom presets are platform-only. The web team owns the ownership filter and labels. [Web integration contract](apps/api/docs/api/settings-usage-web-handoff.md).
