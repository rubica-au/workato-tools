# Xero Practice Manager MCP

The XPM MCP exposes Xero Practice Manager (XPM) API v3.1 operations as callable tools, built on Workato recipe functions. Use it to read and write XPM client data from AI agents and automations.

**API base URL:** `https://api.xero.com/practicemanager/3.1/`\
**Authentication:** Handled automatically via the Workato Xero Practice Manager connection.

---

## Available Tools

### Clients
- [XPM\_Get\_Client\_by\_UUID](#xpm_get_client_by_uuid)
- [XPM\_Query\_Search\_Clients](#xpm_query_search_clients)
- [XPM\_Create\_Client](#xpm_create_client)
- [XPM\_Update\_Client](#xpm_update_client)

### Relationships
- [XPM\_Add\_Relationship](#xpm_add_relationship)
- [XPM\_Update\_Relationship](#xpm_update_relationship)
- [XPM\_Delete\_Relationship](#xpm_delete_relationship)

### Client Groups
- [XPM\_Get\_Client\_Group](#xpm_get_client_group)
- [XPM\_Create\_Client\_Group](#xpm_create_client_group)
- [XPM\_Add\_or\_Remove\_Client\_to\_Group](#xpm_add_or_remove_client_to_group)

---

## Agent Usage Guide

This section describes the recommended call patterns for AI agents working with the XPM MCP.

### Always search before getting or updating

`XPM_Query_Search_Clients` returns a lightweight list with `UUID` and `Name` only. Use it first to find the right client, then call `XPM_Get_Client_by_UUID` to retrieve full details including relationships, contacts, and group membership.

```
Search → Get → Update / Add Relationship / Add to Group
```

### Finding a client's group UUID

A client's group UUID is returned on their full record at `Client.Groups.Group.UUID`. This is the UUID you pass to `XPM_Get_Client_Group` or `XPM_Add_or_Remove_Client_to_Group`. There is no standalone "list all groups" tool -- you need to start from the client record.

### Adding or removing group members

Use `XPM_Add_or_Remove_Client_to_Group` for all group membership changes. You can add and remove in the same call. The tool returns the updated member list so you can confirm the change was applied.

### Note on parameter naming

The tools currently use `client_id` and `group_id` as parameter names in some places, but these fields accept **UUIDs**, not numeric IDs. Always pass the UUID format (e.g. `4773b814-27db-4f18-a05d-f028bd192a6e`). Passing a numeric ID will return an error or empty result.

---

## Clients

### XPM\_Get\_Client\_by\_UUID

Retrieve full details for a single XPM client.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `client_uuid` | string | Yes | UUID of the client to retrieve |

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `Client.UUID` | string | Client UUID |
| `Client.Name` | string | Display name |
| `Client.Email` | string | |
| `Client.Phone` | string | |
| `Client.Address` | string | |
| `Client.City` | string | |
| `Client.Region` | string | |
| `Client.PostCode` | string | |
| `Client.Country` | string | |
| `Client.BusinessStructure` | string | e.g. `Individual`, `Company`, `Trust` |
| `Client.TaxNumber` | string | TFN / IRD |
| `Client.BusinessNumber` | string | ABN |
| `Client.CompanyNumber` | string | ACN / NZBN |
| `Client.GSTRegistered` | string | `Yes` / `No` |
| `Client.IsProspect` | string | `true` / `false` |
| `Client.IsArchived` | string | `true` / `false` |
| `Client.IsDeleted` | string | `true` / `false` |
| `Client.AccountManager.UUID` | string | |
| `Client.AccountManager.Name` | string | |
| `Client.JobManager.UUID` | string | |
| `Client.JobManager.Name` | string | |
| `Client.Groups.Group.UUID` | string | First group UUID (if member of a group) |
| `Client.Groups.Group.Name` | string | First group name |
| `Client.Contacts.Contact[]` | array | Each item has `UUID`, `Name`, `Email`, `Phone`, `Mobile`, `Position`, `IsPrimary` |
| `Client.Relationships.Relationship[]` | array | Each item has `UUID`, `Type`, `RelatedClient`, `NumberOfShares`, `Percentage` |
| `Client.Notes` | string | |
| `Client.WebUrl` | string | |

{% hint style="info" %}
Only the client's **first group** is returned in `Client.Groups.Group`. If a client belongs to multiple groups, retrieve each group separately using its UUID.
{% endhint %}

---

### XPM\_Query\_Search\_Clients

Search XPM clients by name or keyword. Returns a lightweight list -- use `XPM_Get_Client_by_UUID` to retrieve full details for any result.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Name or keyword to search |

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `Clients.Client[]` | array | Each item has `UUID` and `Name` |

**Example response**

```json
{
  "Status": "OK",
  "Clients": {
    "Client": [
      { "UUID": "4773b814-27db-4f18-a05d-f028bd192a6e", "Name": "breed, aaron" },
      { "UUID": "57b1fc33-71d1-45f8-a077-b46786c57af1", "Name": "breed, anita" }
    ]
  }
}
```

---

### XPM\_Create\_Client

Create a new XPM client.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Client display name |
| `email` | string | | |
| `phone` | string | | |
| `address` | string | | Street address |
| `city` | string | | |
| `region` | string | | |
| `post_code` | string | | |
| `country` | string | | |
| `postal_address` | string | | |
| `postal_city` | string | | |
| `postal_region` | string | | |
| `postal_post_code` | string | | |
| `postal_country` | string | | |
| `account_manager_id` | string | | UUID of the account manager |
| `job_manager_id` | string | | UUID of the job manager |
| `billing_client_id` | string | | UUID of the billing client (if different) |
| `first_name` | string | | For individual clients |
| `last_name` | string | | |
| `date_of_birth` | string | | Format: `YYYY-MM-DD` |
| `tax_number` | string | | TFN / IRD |
| `company_number` | string | | ACN / NZBN |
| `business_number` | string | | ABN |
| `business_structure` | string | | `Company`, `Trust`, `Partnership`, `Individual` |
| `prepare_gst` | string | | `Yes` / `No` |
| `gst_registered` | string | | `Yes` / `No` |
| `gst_period` | string | | `1`, `2`, or `6` |
| `gst_basis` | string | | `Invoice`, `Payment`, or `Hybrid` |
| `signed_tax_authority` | string | | `Yes` / `No` |
| `prepare_activity_statement` | string | | `Yes` / `No` |
| `prepare_tax_return` | string | | `Yes` / `No` |
| `referral_source` | string | | |
| `export_code` | string | | |

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `ErrorDescription` | string | Populated on error |
| `Client.UUID` | string | UUID of the newly created client |
| `Client.Name` | string | |

---

### XPM\_Update\_Client

Update an existing XPM client. Only fields provided will be updated.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `client_uuid` | string | Yes | UUID of the client to update |
| `name` | string | Yes | Client display name |
| `email` | string | | |
| `phone` | string | | |
| `address` | string | | |
| `city` | string | | |
| `region` | string | | |
| `post_code` | string | | |
| `country` | string | | |
| `account_manager_id` | string | | UUID of the account manager |
| `job_manager_id` | string | | UUID of the job manager |
| `first_name` | string | | |
| `last_name` | string | | |
| `date_of_birth` | string | | Format: `YYYY-MM-DD` |
| `tax_number` | string | | |
| `company_number` | string | | |
| `business_number` | string | | |
| `business_structure` | string | | |
| `prepare_gst` | string | | `Yes` / `No` |
| `gst_registered` | string | | `Yes` / `No` |
| `gst_period` | string | | `1`, `2`, or `6` |
| `gst_basis` | string | | `Invoice`, `Payment`, or `Hybrid` |
| `signed_tax_authority` | string | | `Yes` / `No` |
| `prepare_activity_statement` | string | | `Yes` / `No` |
| `prepare_tax_return` | string | | `Yes` / `No` |

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `ErrorDescription` | string | Populated on error |

---

## Relationships

### XPM\_Add\_Relationship

Add a relationship between two XPM clients.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `client_uuid` | string | Yes | UUID of the primary client |
| `related_client_uuid` | string | Yes | UUID of the related client |
| `type` | string | Yes | See relationship types below |
| `number_of_shares` | string | | For Shareholder relationships |
| `percentage` | string | | Ownership percentage |
| `start_date` | string | | Format: `YYYY-MM-DD` |
| `end_date` | string | | Format: `YYYY-MM-DD` |

**Relationship type values**

`Director`, `Shareholder`, `Trustee`, `Beneficiary`, `Partner`, `Settlor`, `Associate`, `Secretary`, `Public Officer`, `Husband`, `Wife`, `Spouse`, `Parent Of`, `Child Of`, `Appointer`, `Member`, `Auditor`, `Owner`

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `ErrorDescription` | string | Populated on error |

{% hint style="info" %}
Relationships in XPM are one-directional. Adding a `Spouse` relationship from Client A to Client B does **not** automatically create the reverse. Add both directions if you need the relationship to appear on both client records.
{% endhint %}

---

### XPM\_Update\_Relationship

Update the details of an existing relationship. The relationship UUID is returned on the client record under `Client.Relationships.Relationship[].UUID`.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `relationship_uuid` | string | Yes | UUID of the relationship to update |
| `number_of_shares` | string | | |
| `percentage` | string | | |
| `start_date` | string | | Format: `YYYY-MM-DD` |
| `end_date` | string | | Format: `YYYY-MM-DD` |

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `ErrorDescription` | string | Populated on error |
| `ClientRelationship.UUID` | string | |
| `ClientRelationship.Client.UUID` | string | |
| `ClientRelationship.Client.Name` | string | |
| `ClientRelationship.RelatedClient.UUID` | string | |
| `ClientRelationship.RelatedClient.Name` | string | |
| `ClientRelationship.RelationshipType.Name` | string | |
| `ClientRelationship.NumberOfShares` | string | |
| `ClientRelationship.Percentage` | string | |

---

### XPM\_Delete\_Relationship

Delete an existing relationship by UUID.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `relationship_uuid` | string | Yes | UUID of the relationship to delete |

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `ErrorDescription` | string | Populated on error |

---

## Client Groups

### XPM\_Get\_Client\_Group

Retrieve a client group and its full member list.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `group_uuid` | string | Yes | UUID of the group to retrieve |

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `Group.UUID` | string | |
| `Group.Name` | string | |
| `Group.Taxable` | string | `Yes` / `No` |
| `Group.Clients.Client[]` | array | Each member has `UUID` and `Name` |

**Example response**

```json
{
  "Status": "OK",
  "Group": {
    "UUID": "46dd1d1e-daf9-40f9-a1c2-fd21074978ff",
    "Name": "Breed Family Group",
    "Taxable": "No",
    "Clients": {
      "Client": [
        { "UUID": "4773b814-27db-4f18-a05d-f028bd192a6e", "Name": "Aaron Breed" },
        { "UUID": "57b1fc33-71d1-45f8-a077-b46786c57af1", "Name": "breed, anita" }
      ]
    }
  }
}
```

{% hint style="warning" %}
The group UUID is not directly searchable. Retrieve it from a client record at `Client.Groups.Group.UUID` after calling `XPM_Get_Client_by_UUID`.
{% endhint %}

---

### XPM\_Create\_Client\_Group

Create a new client group, optionally with an initial member.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | string | Yes | Group name |
| `taxable` | string | | `Yes` / `No` (default `No`) |
| `client_uuid` | string | | UUID of the first client to add |

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `ErrorDescription` | string | Populated on error |
| `Group.UUID` | string | UUID of the newly created group |
| `Group.Name` | string | |
| `Group.Taxable` | string | |
| `Group.Clients.Client[]` | array | Members, each with `UUID` and `Name` |

---

### XPM\_Add\_or\_Remove\_Client\_to\_Group

Add and/or remove a client from a group in a single call. At least one of `add_client_uuid` or `remove_client_uuid` must be provided. Returns the updated member list.

**Input**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `group_uuid` | string | Yes | UUID of the group |
| `add_client_uuid` | string | | UUID of the client to add |
| `remove_client_uuid` | string | | UUID of the client to remove |

**Response**

| Field | Type | Description |
|-------|------|-------------|
| `Status` | string | `OK` or `ERROR` |
| `ErrorDescription` | string | Populated on error |
| `Group.UUID` | string | |
| `Group.Name` | string | |
| `Group.Taxable` | string | |
| `Group.Clients.Client[]` | array | Updated member list, each with `UUID` and `Name` |

---

## Error Handling

All tools return a `Status` field. On failure, `Status` will be `ERROR` and `ErrorDescription` will contain the XPM error message. Always check `Status == "OK"` before using response data in downstream steps.

**Common errors**

| Error | Likely cause |
|-------|-------------|
| `Status: ERROR` with empty `ErrorDescription` | A numeric ID was passed where a UUID is expected |
| Empty `Clients.Client[]` on search | No match found -- try a shorter or different keyword |
| `Group.Clients` empty on group get | The group UUID was correct but no members are assigned |

---

## Known Gaps

The following XPM API operations are not yet implemented:

| Operation | XPM Endpoint |
|-----------|-------------|
| List all client groups | `GET clientgroup.api/list` |
| Update a client group | `PUT clientgroup.api` |
| Delete a client group | `DELETE clientgroup.api/{uuid}` |
| Delete a client | `DELETE client.api/{uuid}` |
