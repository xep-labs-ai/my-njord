# Resource API Specification

## Purpose

This document defines the HTTP contract for resource CRUD and daily snapshot ingestion for v1.

It specifies endpoint paths, request/response shapes, writable fields, ingestion validation rules, and error cases for StorageHotel and VirtualMachine resources.

Domain models and billing semantics remain in the resource-specific PRPs.

---

## Common Rules

All endpoints are under `/api/v1/`.

Request and response bodies use JSON.

`billing_account` is the integer primary key of BillingAccount. Resources must belong to the specified billing account for all write operations.

---

## StorageHotel Endpoints

### POST /api/v1/storage-hotels/

Create a new StorageHotel resource.

**Writable fields:**

- `name` (CharField, required)
- `filesystem_identifier` (CharField, required)
- `quota_unit` (CharField, required) — values: `"KB"`, `"KIB"`
- `billing_account` (integer PK, optional, nullable)
- `active_from` (date, required)
- `active_to` (date, optional)

**Read-only fields:**

- `id`
- `status` (defaults to `UNASSIGNED`)
- `created_at`
- `updated_at`
- `deleted_at`

**Request example:**

```json
{
  "name": "storage-primary",
  "filesystem_identifier": "/mnt/storage-primary",
  "quota_unit": "KIB",
  "billing_account": 1,
  "active_from": "2026-01-01",
  "active_to": null
}
```

**Response (201):**

```json
{
  "id": 101,
  "name": "storage-primary",
  "filesystem_identifier": "/mnt/storage-primary",
  "quota_unit": "KIB",
  "billing_account": 1,
  "status": "UNASSIGNED",
  "active_from": "2026-01-01",
  "active_to": null,
  "created_at": "2026-01-15T10:00:00Z",
  "updated_at": "2026-01-15T10:00:00Z",
  "deleted_at": null
}
```

---

### GET /api/v1/storage-hotels/

List StorageHotel resources.

**Query parameters:**

- `billing_account` (optional, integer) — filter by billing account
- `status` (optional, string) — filter by status (UNASSIGNED, ACTIVE, RETIRED)
- `active_from` (optional, date) — filter resources active on or after this date
- `active_to` (optional, date) — filter resources active on or before this date

**Response (200):**

```json
{
  "count": 10,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 101,
      "name": "storage-primary",
      "filesystem_identifier": "/mnt/storage-primary",
      "quota_unit": "KIB",
      "billing_account": 1,
      "status": "ACTIVE",
      "active_from": "2026-01-01",
      "active_to": null,
      "created_at": "2026-01-15T10:00:00Z",
      "updated_at": "2026-01-15T10:00:00Z",
      "deleted_at": null
    }
  ]
}
```

---

### GET /api/v1/storage-hotels/{id}/

Retrieve a single StorageHotel resource.

**Response (200):**

Same structure as POST response.

---

### PATCH /api/v1/storage-hotels/{id}/

Partially update a StorageHotel resource.

**Patchable fields:**

- `name`
- `billing_account`
- `active_from`
- `active_to`
- `status`
- `quota_unit` — patchable with correction semantics (see warning below)

**Not patchable after creation:**

- `filesystem_identifier` — functions as the StorageHotel external/natural identifier and must remain stable

**`quota_unit` correction warning:** Changing `quota_unit` is allowed in v1 as an administrative correction workflow, but it is a high-impact change because it alters how `quota_raw` values are interpreted for billing. Because historical unit assignment is not tracked, changing `quota_unit` after snapshots already exist may affect invoice generation for uninvoiced historical periods. Operational rule: if `quota_unit` is changed, affected draft invoices should be regenerated before finalization. Already finalized invoices remain immutable. The change should be audit-logged.

**Request example:**

```json
{
  "status": "ACTIVE",
  "active_to": "2026-12-31"
}
```

**Response (200):**

Updated resource representation.

---

### POST /api/v1/storage-hotels/{id}/quota

Ingest a daily quota snapshot.

**Request body:**

```json
{
  "date": "2026-01-15",
  "quota_raw": "5000"
}
```

**Response (201):**

```json
{
  "id": 1001,
  "storage_hotel": 101,
  "date": "2026-01-15",
  "quota_raw": "5000",
  "created_at": "2026-01-15T10:30:00Z"
}
```

**Validation rules:**

- `date` must be <= today (no future-dated snapshots)
- `quota_raw` must be a decimal number >= 0 with precision constraints: `max_digits=25, decimal_places=4`. This represents the raw quota value in the unit specified by `quota_unit` (KB or KiB); during billing it is normalized to TB for pricing calculations. Note: `quota_raw = 0` is explicitly valid. It produces `normalized_usage = {"quota_tb": "0"}` and `daily_cost = 0`.
- If a snapshot already exists for the same storage hotel and date, return 409 Conflict — no silent overwrite

---

## VirtualMachine Endpoints

### POST /api/v1/virtual-machines/

Create a new VirtualMachine resource.

**Writable fields:**

- `name` (CharField, required)
- `provisioner` (CharField, required) — values: `"VCENTER"`
- `billing_account` (integer PK, optional, nullable)
- `active_from` (date, required)
- `active_to` (date, optional)

**Read-only fields:**

- `id`
- `status` (defaults to `UNASSIGNED`)
- `created_at`
- `updated_at`
- `deleted_at`

**Request example:**

```json
{
  "name": "vm-compute-01",
  "provisioner": "VCENTER",
  "billing_account": 1,
  "active_from": "2026-01-01",
  "active_to": null
}
```

**Response (201):**

```json
{
  "id": 205,
  "name": "vm-compute-01",
  "provisioner": "VCENTER",
  "billing_account": 1,
  "status": "UNASSIGNED",
  "active_from": "2026-01-01",
  "active_to": null,
  "created_at": "2026-01-15T10:00:00Z",
  "updated_at": "2026-01-15T10:00:00Z",
  "deleted_at": null
}
```

---

### GET /api/v1/virtual-machines/

List VirtualMachine resources.

**Query parameters:**

- `billing_account` (optional, integer) — filter by billing account
- `status` (optional, string) — filter by status (UNASSIGNED, ACTIVE, RETIRED)
- `provisioner` (optional, string) — filter by provisioner

**Response (200):** Paginated list.

Example:

```json
{
  "count": 5,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 205,
      "name": "vm-compute-01",
      "provisioner": "VCENTER",
      "billing_account": 1,
      "status": "ACTIVE",
      "active_from": "2026-01-01",
      "active_to": null,
      "created_at": "2026-01-15T10:00:00Z",
      "updated_at": "2026-01-15T10:00:00Z",
      "deleted_at": null
    }
  ]
}
```

---

### GET /api/v1/virtual-machines/{id}/

Retrieve a single VirtualMachine resource.

**Response (200):**

Same structure as POST response.

---

### PATCH /api/v1/virtual-machines/{id}/

Partially update a VirtualMachine resource.

**Patchable fields:**

- `name`
- `billing_account`
- `active_from`
- `active_to`
- `status`

**Not patchable after creation:**

- `provisioner` — part of the VM identity/context and must remain stable

**Request example:**

```json
{
  "status": "ACTIVE"
}
```

**Response (200):**

Updated resource representation.

---

### POST /api/v1/virtual-machines/{id}/usage

Ingest a daily usage snapshot.

**Request body:**

```json
{
  "date": "2026-01-15",
  "cpu_count": "8",
  "ram_mb": "65536",
  "disks_total_gb": "500"
}
```

**Response (201):**

```json
{
  "id": 2001,
  "virtual_machine": 205,
  "date": "2026-01-15",
  "cpu_count": "8",
  "ram_mb": "65536",
  "disks_total_gb": "500",
  "created_at": "2026-01-15T10:30:00Z"
}
```

**Validation rules:**

- `date` must be <= today (no future-dated snapshots)
- `cpu_count`, `ram_mb`, `disks_total_gb` must be non-negative numbers. Note: zero values are explicitly valid for any dimension and produce `daily_cost = 0` for that dimension, consistent with the StorageHotel zero-quota behavior.
- If a snapshot already exists for the same virtual machine and date, return 409 Conflict — no silent overwrite

---

## Ingestion Rules

### Duplicate Snapshot Behavior

If a snapshot already exists for the same resource and date, ingestion fails with **409 Conflict** using error code `duplicate_snapshot`. No silent overwrite occurs.

Overwrite and correction mechanisms, if needed later, must be defined explicitly.

### Future-Date Policy

Snapshots with dates in the future (> today) are rejected with **400 Bad Request**.

All snapshots must represent historical or current-day data.

### Validation Failures

Common error responses for all endpoints:

- **400 Bad Request** — validation error (invalid date format, negative values, unknown provisioner, etc.)
- **404 Not Found** — resource not found
- **409 Conflict** — duplicate snapshot for same resource and date
- **422 Unprocessable Entity** — business rule violation (e.g., billing account mismatch)

---

## Resource Status Lifecycle Transitions

Resources are created with `status = UNASSIGNED`. Status transitions happen via PATCH.

Allowed transitions:

- `UNASSIGNED` → `ACTIVE` — requires `billing_account` and `active_from` to be set
- `ACTIVE` → `RETIRED` — must set `active_to` in the same PATCH
- `UNASSIGNED` → `RETIRED` — **not allowed**. Resources that were created by mistake must be corrected through other means (e.g., soft-delete or administrative cleanup). This transition is prohibited because `active_from` is required on `ResourceModel` and cannot be null, making a direct `UNASSIGNED → RETIRED` transition inconsistent with the model definition.
- `RETIRED` → `ACTIVE` — **not allowed** (prevents accidental rebilling)

`billing_account` is optional on resource creation. A resource can be created without a billing account and remain `UNASSIGNED` until assigned.

When patching `status = RETIRED` from `ACTIVE`, the request must include `active_to`. If `active_to` is missing, the request returns 400.

**Validation: `active_to >= active_from`**

If both `active_from` and `active_to` are set, `active_to` must be >= `active_from`. If `active_to < active_from`, the request returns 400.

When a resource transitions to `RETIRED` via PATCH, the service layer sets `deleted_at` to the current timestamp automatically. There is no dedicated DELETE endpoint in v1.

Changing `active_from` or `active_to` on a resource that has been included in a finalized invoice is allowed. Finalized invoices are immutable and unaffected. The change only affects future invoice generation runs.

**v1 limitation — billing account reassignment:**

> If a resource's `billing_account` is changed after historical usage has been captured, invoice generation for past uninvoiced periods will use the current `billing_account`, not historical ownership. Previously generated draft invoices for affected periods should be regenerated before finalization. Already finalized invoices remain immutable. This limitation should be revisited in a future version if reassignment becomes frequent enough to affect billing correctness materially.

**v1 limitation — single-record ingestion:**

> v1 ingestion endpoints accept one snapshot per request. At ~10,000 resources, this means ~10,000 API calls per day. Bulk ingestion is a likely future enhancement.

---
