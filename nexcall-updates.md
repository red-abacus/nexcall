# NexCall backend updates

## Update #1: Introduce the APIs required for the Settings section

### Context

The Settings UI currently has no persistent backend data. Industry classifiers are read from bundled JSON files, while the Twilio-number inventory is supplied by an in-memory HTTP interceptor and is reset on browser reload. The backend must replace those prototypes with persistent APIs; the UI will move to the resulting classifier contract.

### Goal

Provide persistent, authorized CRUD APIs for:

- standardised industry classifications;
- NexCall's existing Twilio calling-number inventory and its tenant assignment;
- returning the calling numbers that a user may select for a campaign.

This is inventory management only. It does not buy, provision, release, or reconfigure numbers in Twilio.

### Details

#### Industry classifiers

Canonical resource: `/api/settings/industry-classifiers`. Each classifier is one standardised entry:

```json
{ "code": "6210", "description": "Custom software development", "standard": "CAEN_REV3" }
```

`standard` identifies the published classification set, including its version; for example, `CAEN_REV3` and `NAICS_US_2022`. `code` remains a string so leading zeroes are preserved. The combination of `standard` and `code` is unique.

| Request | Purpose |
| --- | --- |
| `GET /api/settings/industry-classifiers` | List classifiers; `standard` is an optional query parameter. |
| `GET /api/settings/industry-classifiers?standard=CAEN_REV3` | List classifiers in one standard. |
| `POST /api/settings/industry-classifiers` | Create `{ code, description, standard }`. |
| `GET /api/settings/industry-classifiers/{standard}/{code}` | Read one classifier. |
| `PATCH /api/settings/industry-classifiers/{standard}/{code}` | Update its description. |
| `DELETE /api/settings/industry-classifiers/{standard}/{code}` | Delete an unreferenced classifier. |

Seed the existing CAEN Rev. 3 and NAICS US 2022 data, and make this resource the single source of truth for Settings and CRM. Reject duplicate entries and prevent deletion of entries used by CRM records. The current UI catalog routes will need to move to this simplified contract; no separate CRM-owned classifier data should be created.

#### Twilio-number inventory

Base resource: `/api/settings/twilio-numbers`.

| Request | Purpose |
| --- | --- |
| `GET /api/settings/twilio-numbers` | List accessible inventory records. Supports `tenantId`, `availableToTenantId`, and `includeUnassigned` filters. |
| `POST /api/settings/twilio-numbers` | Create an unassigned number: `{ phoneNumber }`. |
| `GET /api/settings/twilio-numbers/{id}` | Read one record. |
| `PATCH /api/settings/twilio-numbers/{id}` | Update the phone number: `{ phoneNumber }`; do not change its assignment. |
| `DELETE /api/settings/twilio-numbers/{id}` | Delete an unassigned record. |
| `PATCH /api/settings/twilio-numbers/{id}/tenant` | Assign or release a number: `{ tenantId, expectedTenantId }`. `tenantId: null` releases it. |

Return the same shape for each inventory record:

```json
{
  "id": 1,
  "phoneNumber": "+19069702512",
  "tenantId": null,
  "tenantName": null
}
```

`phoneNumber` must be normalized as an international E.164 number and unique. A number can belong to zero or one tenant; a tenant can have multiple numbers. Resolve `tenantName` from the tenant record rather than accepting it from the client.

Platform administrators may create, edit, delete, assign, and release inventory records. Tenant users may read and use only numbers assigned to their tenant. Platform administrators may also use an unassigned pool number for a campaign. Enforce these rules on the server; UI guards and query parameters are not authorization.

Treat assignment as an atomic update. `expectedTenantId` represents the assignment the client last saw; return `409 Conflict` if it has changed. Also return `400` for invalid phone numbers or assignments, `404` for missing numbers or tenants, and `409` for duplicate numbers or attempts to delete an assigned number.

The existing campaign `twilioNumber` value must remain compatible. On campaign create or update, validate that the selected number is in the inventory and is eligible for that campaign's tenant and caller. Do not retroactively rewrite existing campaigns or live calls when an inventory record changes.

## Update #2: Extend customers (tenants) with billing info and Twilio numbers

### Context

The UI now calls customers **tenants**, but it continues to use the existing `/api/customers` API. Its tenant data is currently mock-backed and has basic contact fields plus optional demonstration billing data. The backend must make this information persistent and expose the Twilio numbers allocated to each tenant.

### Goal

Extend the existing customer resource so it represents a tenant with required contact and billing information. Keep `/api/customers` as the API path and existing customer operations intact. Make each tenant's assigned Twilio numbers visible and manageable through the Settings inventory introduced in Update #1.

### Details

#### Customer / tenant data

Continue using `/api/customers`; use **tenant** in Swagger titles, descriptions, and user-facing documentation. Existing client integrations may still use the customer name and URL.

Each tenant must have these required fields:

```json
{
  "name": "Example Tenant",
  "address": "1 Example Street",
  "email": "admin@example.com",
  "phoneNo": "+40123456789",
  "billingDetails": {
    "entityName": "Example Tenant SRL",
    "businessId": "RO12345678",
    "tradeRegistryId": "J40/123/2026",
    "bankAccount": "RO00BANK0000000000000000",
    "bankName": "Example Bank",
    "websiteUrl": "https://example.com"
  }
}
```

`billingDetails` is required, and all fields shown above are required. Validate email, phone number, URL, and non-empty billing values on create and update. Return the complete `billingDetails` object in customer list and detail responses. Existing tenants without billing data must be migrated or completed before they can be treated as valid tenants.

Keep the current customer CRUD endpoints and payload conventions:

| Request | Purpose |
| --- | --- |
| `GET /api/customers` | List tenants. |
| `POST /api/customers` | Create a tenant with required contact and billing data. |
| `GET /api/customers/{id}` | Read a tenant, including billing data and assigned Twilio numbers. |
| `PATCH /api/customers?customerId={id}` | Update contact and billing data. |
| `DELETE /api/customers/{id}` | Delete a tenant only when related data, including assigned numbers, has been handled safely. |

#### Assigned Twilio numbers

A tenant can have multiple Twilio numbers; a number can have at most one tenant. Do not add a single `twilioNumber` column to the customer record. The number inventory and ownership relationship remain owned by the Settings API from Update #1.

Include an `assignedTwilioNumbers` array in the customer detail response, using the existing number shape:

```json
"assignedTwilioNumbers": [
  { "id": 1, "phoneNumber": "+19069702512", "tenantId": 42, "tenantName": "Example Tenant" }
]
```

To add, reassign, or release a number for a tenant, use `PATCH /api/settings/twilio-numbers/{id}/tenant` with `{ tenantId, expectedTenantId }`. The server must validate that the tenant exists and perform the change atomically. Do not delete a tenant while numbers are assigned; require those numbers to be released or reassigned first.

## Update #3: Update CRM to use industry classifiers

### Context

CRM accounts currently use mock industry values. The industry must instead be selected from the standardised classifier data introduced in Update #1, so the code and description are consistent across CRM records.

### Goal

Allow each CRM account to be classified by an industry standard and code. The backend must validate the selection against the Settings classifier API and return the matching description.

### Details

Add these fields to the CRM account create, update, and read contracts:

```json
{
  "industryStandard": "CAEN_REV3",
  "industryCode": "6210",
  "industryDescription": "Custom software development"
}
```

Clients provide `industryStandard` and `industryCode`. The server verifies that the pair exists in `/api/settings/industry-classifiers`, then resolves and returns `industryDescription`; clients must not set the description themselves. Reject an unknown or mismatched standard/code pair with `400 Bad Request`.

Keep the classification fields on CRM accounts, not on contacts. The CRM UI can query `GET /api/settings/industry-classifiers?standard={standard}` to populate the industry selector. Existing mock-only classifier fields (`industryClassifierId`, `industryClassifierName`, and `industryVersion`) should be replaced by this simpler standard-and-code contract.
