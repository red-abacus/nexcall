# NexCall backend updates

## Update #1: Introduce the APIs required for the Settings section

### Context

The Settings UI currently has no persistent backend data. Industry classifiers are read from bundled JSON files, while the Twilio-number inventory is supplied by an in-memory HTTP interceptor and is reset on browser reload. The backend must replace those prototypes without changing the UI's existing contracts.

### Goal

Provide persistent, authorized CRUD APIs for:

- industry classifiers and their entries;
- NexCall's existing Twilio calling-number inventory and its tenant assignment;
- returning the calling numbers that a user may select for a campaign.

This is inventory management only. It does not buy, provision, release, or reconfigure numbers in Twilio.

### Details

#### Industry classifiers

Canonical resource: `/api/settings/industry-classifiers`.

| Request | Purpose |
| --- | --- |
| `GET /api/settings/industry-classifiers` | List the catalog as `{ classifiers, countries }`. |
| `POST /api/settings/industry-classifiers` | Create a classifier and its metadata. |
| `GET /api/settings/industry-classifiers/{classifierId}` | Read one classifier. |
| `PATCH /api/settings/industry-classifiers/{classifierId}` | Update classifier metadata. |
| `DELETE /api/settings/industry-classifiers/{classifierId}` | Delete an unreferenced classifier. |
| `GET /api/settings/industry-classifiers/{classifierId}/entries` | List its entries. |
| `POST /api/settings/industry-classifiers/{classifierId}/entries` | Create an entry. |
| `PATCH /api/settings/industry-classifiers/{classifierId}/entries/{code}` | Update an entry. |
| `DELETE /api/settings/industry-classifiers/{classifierId}/entries/{code}` | Delete an unreferenced entry. |

`classifiers` contains `{ id, name, description, version, countryCodes, language, sourceUrl }`; `countries` contains `{ code, name }`; and each entry contains `{ code, name, description }`. Seed the existing CAEN Rev. 3 and NAICS 2022 data, preserve codes as strings, and make the catalog the shared source of truth for Settings and future CRM work. A classifier ID, version, and entry code identify the catalog data and should not be silently changed; reject duplicates and prevent deletion when a classifier or entry is in use.

The current UI reads `/api/crm/industry-classifiers` and `/api/crm/industry-classifiers/{classifierId}/entries`. Keep these two `GET` routes as read-only compatibility aliases backed by the same Settings data until the UI is changed to the canonical Settings routes.

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
