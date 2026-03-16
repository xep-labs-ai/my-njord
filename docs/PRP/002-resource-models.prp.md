# Resource Models

Defines all **billable resource models**.

---

# BillingAccountBase (Abstract)

An abstract Django model defining generic account identity and contact information.

Fields:

```
name
contact_point
contact_email
contact_telephone_number
customer_number
make_invoice
internal_customer
```

Rules:

- `name` (CharField, required)
- `contact_point` (CharField, optional, blank=True, null=True)
- `contact_email` (EmailField, optional, blank=True, null=True)
- `contact_telephone_number` (CharField, optional, blank=True, null=True)
- `customer_number` (CharField, optional, unique when set, blank=True, null=True)
- `make_invoice` (BooleanField, default=True) — controls whether invoice generation runs for this account. Resources belonging to an account where `make_invoice = False` are excluded from all invoice generation runs. The `POST /api/v1/invoices/generate` endpoint returns 422 (`billing_account_not_billable`) if the billing account has `make_invoice = False`.
- `internal_customer` (BooleanField, default=True)

Purpose:

This abstract base allows implementations in different environments to extend generic account fields without carrying UiO-specific fields.

---

# PriceList

Defines pricing for a BillingAccount.

Fields:

```
id
name
created_at
updated_at
```

Rules:

- `id` (integer PK)
- `name` (CharField, required, unique)
- `created_at` / `updated_at`
- No status field in v1 — PriceLists are always active
- Price validity is controlled by `ResourcePrice.effective_from` / `ResourcePrice.effective_to`

---

# BillingAccount (UiO Implementation)

Concrete account model for UiO / USIT environment.

Fields:

```
id
name
price_list
contact_point
contact_email
contact_telephone_number
customer_number
make_invoice
internal_customer
usit_contact_point
main_agreement_id
main_agreement_description
usit_accounting_place
usit_sub_project
ephorte
uio_unit
created_at
updated_at
```

Base fields (inherited from BillingAccountBase):

- `name`
- `contact_point`
- `contact_email`
- `contact_telephone_number`
- `customer_number`
- `make_invoice`
- `internal_customer`

UiO-specific fields:

- `usit_contact_point` — operational contact inside USIT
- `main_agreement_id` — reference to the primary service agreement
- `main_agreement_description` — description of the primary service agreement
- `usit_accounting_place` — accounting classification for internal financial reporting
- `usit_sub_project` — accounting classification for internal financial reporting
- `ephorte` — optional reference to UiO document archive / case system
- `uio_unit` — UiO organizational unit identifier

Rules:

- `id` (integer PK)
- `name` (CharField, required, unique)
- `price_list` (FK to PriceList, required, on_delete=PROTECT) — each BillingAccount must use exactly one PriceList
- All UiO-specific fields: optional (blank=True, null=True)
- `created_at` / `updated_at`

---

# Base Models

## TimestampedModel

Used for **mutable models**.

Fields:

```
created_at
updated_at
```

---

## CreatedAtModel

Used for **append-only models**.

Fields:

```
created_at
```

---

# ResourceModel

Base model for billable resources.

This is a **Django abstract model** (`abstract = True`). The `resource_type + resource_id` pattern on InvoiceLine and InvoiceDailyCost is the correct cross-resource reference strategy.

## Managers

### Default manager

The default manager excludes soft-deleted resources (those with `deleted_at` set).

### `billing_objects` manager

A dedicated manager for the billing engine that **includes soft-deleted resources**. This is necessary because the billing engine must be able to bill historical periods for resources that have since been soft-deleted. The inclusion of a resource via `billing_objects` does not make it billable — per-day billability rules still apply (determined by `active_from`, `active_to`, and billing account assignment).

All resource types that inherit from `ResourceModel` must have this manager available.

Fields:

```
billing_account
name
status
active_from
active_to
deleted_at
```

- `deleted_at` — DateTimeField, nullable (null = not soft-deleted). Used for soft-delete semantics across all resource types.

Field types:

- `billing_account` — FK to BillingAccount, nullable (null = unassigned), on_delete=PROTECT
- `name` — CharField(max_length=255), required, not unique
- `status` — CharField(max_length=20, choices=["UNASSIGNED", "ACTIVE", "RETIRED"], default="UNASSIGNED")
- `active_from` (date, required): the first day the resource is billable. For resources starting in UNASSIGNED status, set `active_from` to the intended billing start date. Activating a resource does not automatically update `active_from`. Setting `active_from` far in the past will create a backdated billing window.
- `active_to` (date, nullable): the last day the resource is billable. `null` means open-ended (no end date).

Status lifecycle:

```
UNASSIGNED
ACTIVE
RETIRED
```

**Status transition rules:**

| From | To | Allowed | Reason |
|---|---|---|---|
| `UNASSIGNED` | `ACTIVE` | Yes | Requires `billing_account` and `active_from` to be set |
| `ACTIVE` | `RETIRED` | Yes | Must set `active_to` in the same operation |
| `RETIRED` | `ACTIVE` | **No** | Prevents accidental rebilling of retired resources |
| `UNASSIGNED` | `RETIRED` | **No** | `active_from` is required on `ResourceModel` and cannot be null, making a direct `UNASSIGNED -> RETIRED` transition inconsistent with the model definition |

See `004-resource-api.prp.md` for the HTTP-level enforcement of these transitions.

Validation rules:

- `active_to >= active_from` must be enforced at the service layer **and** as a PostgreSQL DB check constraint. A resource with `active_from > active_to` is invalid.

Per-day billability rule:

A resource is billable for a given day only if:

- `billing_account IS NOT NULL`
- `active_from <= day`
- `active_to IS NULL OR day <= active_to`

If `status == RETIRED`, `active_to` should be set to the final billable day.

**Patching `active_from`/`active_to` after finalized invoices:**

Changing `active_from` or `active_to` on a resource that has been included in a finalized invoice is allowed. Finalized invoices are immutable by design and represent a point-in-time calculation. The change only affects future invoice generation runs.

---

# StorageHotel

Represents a **filesystem resource**.

All fields (including inherited from ResourceModel):

```
billing_account
filesystem_identifier
quota_unit
status
active_from
active_to
created_at
updated_at
```

Quota units:

```
KB
KIB
```

Constraint:

```
filesystem_identifier UNIQUE
```

Soft-delete semantics:

- If `deleted_at` is set, `status` must be `RETIRED`
- If `deleted_at` is set, `active_to` must be set
- `active_to` must be on or before the calendar date of `deleted_at`
- Default querysets exclude soft-deleted resources; billing and audit workflows can access them
- Billability for historical days is resolved from `active_from`/`active_to`, not from `deleted_at` alone

---

# StorageHotelDailyQuota

Daily quota snapshot.

Fields:

```
storage_hotel
date
quota_raw
created_at
```

Field types:

- `storage_hotel` — FK to StorageHotel, required, on_delete=CASCADE
- `quota_raw` — DecimalField(max_digits=25, decimal_places=4)

Constraint:

```
(storage_hotel_id, date) UNIQUE
```

---

# QuotaIngestionEvent

Audit log for quota ingestion.

Fields:

```
storage_hotel
date
raw_payload
normalized_quota_raw
request_id
created_at
```

---

# VirtualMachine

Represents a compute resource.

All fields (including inherited from ResourceModel):

```
billing_account
name
status
provisioner
active_from
active_to
created_at
updated_at
```

Provisioners:

```
VCENTER
```

**v1 uniqueness note:** There is no natural-key uniqueness constraint in v1 beyond the primary key. Callers and ingestion workflows are responsible for avoiding duplicate VM creation. Duplicate VirtualMachine rows, if created, are treated as separate billable resources.

Soft-delete semantics:

- If `deleted_at` is set, `status` must be `RETIRED`
- If `deleted_at` is set, `active_to` must be set
- `active_to` must be on or before the calendar date of `deleted_at`
- Default querysets exclude soft-deleted resources; billing and audit workflows can access them
- Billability for historical days is resolved from `active_from`/`active_to`, not from `deleted_at` alone

---

# VirtualMachineDailyUsage

Daily usage snapshot.

Fields:

```
virtual_machine
date
cpu_count
ram_mb
disks_total_gb
created_at
```

Field types:

- `virtual_machine` — FK to VirtualMachine, required, on_delete=CASCADE
- `cpu_count` — PositiveIntegerField
- `ram_mb` — DecimalField(max_digits=14, decimal_places=2)
- `disks_total_gb` — DecimalField(max_digits=14, decimal_places=2)

Constraint:

```
(virtual_machine_id, date) UNIQUE
```

---

# VirtualMachineUsageIngestionEvent

Audit log for VM usage ingestion.

Fields:

```
virtual_machine
date
raw_payload
normalized_cpu_count
normalized_ram_mb
normalized_disks_total_gb
request_id
created_at
```

---

# Invoice

Fields:

```
id
invoice_number
billing_account
period_start
period_end
currency
status
total_amount
incomplete
selection_scope
selection_fingerprint
created_at
updated_at
finalized_at
metadata
```

- `incomplete` — BooleanField(default=False). `true` when `force=true` was used and at least one resource/day could not be resolved from a real snapshot or successful autofill, and the system used fallback behavior (e.g., zero-cost billing). `false` when all resource-days were resolved from real snapshots or successful carry-forward autofill. This is a dedicated database column, not a metadata-only flag, to support direct filtering in list queries.

Metadata may include:
```
selected_resource_types
explicit_resources
force
autofill_missing_days
provisional
missing_data_summary
```

Note: `selection_scope` is a top-level database field on `Invoice` and does not need to be duplicated in `metadata`.

Metadata flag semantics:

- `provisional` (boolean) — `true` when `period_end > today` at generation time; the invoice covers future days filled via autofill. Defaults to `false`.
- `missing_data_summary` — present when `incomplete=true`; reports which resources had missing days.

Uniqueness constraint:

There must be at most one draft invoice per `(billing_account, period_start, period_end, selection_scope, selection_fingerprint)`.

A matching finalized invoice must block regeneration entirely (finalized invoices are immutable).

A matching draft is replaced atomically when `force=true`.

Invoice number assignment:

`invoice_number` is `null` on draft invoices. It is assigned during finalization and is immutable once set.

Field types:

- `invoice_number` — CharField(max_length=20), nullable, unique when set (fits `INV-YYYY-mm-NNNNN`)
- `billing_account` — FK to BillingAccount, required, on_delete=PROTECT
- `period_start` — DateField, required
- `period_end` — DateField, required
- `currency` — CharField(max_length=3, default="NOK")
- `selection_scope` — CharField(max_length=50), required — identifies the selection category (e.g., "all_resources", "resource_types", "explicit_resources")
- `selection_fingerprint` — CharField(max_length=64), required — SHA-256 lowercase hex digest of the canonical selection payload
- `status` — CharField(max_length=20, choices=["draft", "finalized"], default="draft")
- `total_amount` — DecimalField(max_digits=12, decimal_places=2), nullable — null only before generation runs; set during draft creation and updated on recalculation
- `metadata` — JSONField(default=dict)
- `finalized_at` — DateTimeField, nullable
- `created_at` / `updated_at` — DateTimeField, auto

## Currency Consistency Rule

`InvoiceLine.currency` and `InvoiceDailyCost.currency` must match `Invoice.currency`. This is a service-layer invariant and may be enforced by a database check constraint.


---

# InvoiceLine

Represents billing aggregation for **one resource within an invoice**.

Fields:

```
id
invoice
resource_type
resource_id
description
total_cost
currency
metadata
```

Metadata must include `billing_dimensions` and `total_quantity_by_dimension`.

**Naming convention for `total_quantity_by_dimension` keys:** Keys use the format `{pricing_dimension}_days`, representing the cumulative quantity-days for that dimension across all billed days in the invoice period. Examples: `quota_tb_days`, `cpu_count_days`, `ram_gb_days`, `disk_gb_days`.

Standard metadata structure for StorageHotel:

```json
{
  "billing_dimensions": ["quota_tb"],
  "total_quantity_by_dimension": {
    "quota_tb_days": "3720"
  },
  "quota_unit": "KB",
  "resource_snapshot": {
    "id": 101,
    "name": "storage-primary",
    "filesystem_identifier": "/mnt/storage-primary",
    "quota_unit": "KB"
  }
}
```

Standard metadata structure for VirtualMachine:

```json
{
  "billing_dimensions": ["cpu_count", "ram_gb", "disk_gb"],
  "total_quantity_by_dimension": {
    "cpu_count_days": "248",
    "ram_gb_days": "992",
    "disk_gb_days": "15500"
  },
  "provisioner": "VCENTER",
  "resource_snapshot": {
    "id": 205,
    "name": "vm-prod-001",
    "provisioner": "VCENTER"
  }
}
```

Metadata must also include:

- `resource_snapshot` — **required** for all resources. Frozen snapshot of the resource's identifying attributes at generation time (name, id, and resource-type-specific fields). Required for audit reproducibility.

Resource-type-specific required fields in metadata:

- `quota_unit` — required for `resource_type = "storage_hotel"`. Needed to verify unit conversion at audit time.
- `provisioner` — required for `resource_type = "virtual_machine"`. Records the provisioner at billing time.

**Intentional metadata redundancy:** `quota_unit` (StorageHotel) and `provisioner` (VirtualMachine) intentionally appear both as flat top-level keys in `InvoiceLine.metadata` AND inside `resource_snapshot`. The flat top-level key provides quick access without nested lookup; the copy inside `resource_snapshot` ensures the snapshot is self-contained for audit purposes.

Metadata may also include:
```
price_summary
```

Multi-dimension resources:

For multi-dimension resources (e.g. VirtualMachine), one InvoiceLine is created per resource. The `total_cost` is the sum of all `InvoiceDailyCost` rows for that resource across all dimensions and all days in the billing period.

Field types:

- `invoice` — FK to Invoice, on_delete=CASCADE, required
- `resource_type` — CharField(max_length=50), required
- `resource_id` — PositiveIntegerField, required
- `description` — CharField(max_length=255), optional (blank=True, null=True) — frozen at invoice generation time; set to the resource's `name` field if present and non-blank, otherwise falls back to `{ResourceType} #{resource_id}` (e.g., `StorageHotel #101`). Never recomputed from the live resource after generation.
- `total_cost` — DecimalField(max_digits=14, decimal_places=10), required (10 decimal places, not rounded)
- `currency` — CharField(max_length=3, default="NOK")
- `metadata` — JSONField(default=dict)

---

# InvoiceDailyCost

Daily billing breakdown.

There is **one `InvoiceDailyCost` row per resource per day**. For multi-dimension resources (e.g., VirtualMachine), the per-dimension usage and costs are stored in the row's `metadata` under `normalized_usage`, `resolved_prices`, and `dimension_costs`.

Fields:

```
id
invoice
resource_type
resource_id
date
daily_cost
currency
autofilled
metadata
created_at
```

Constraint:
(invoice_id, resource_type, resource_id, date) UNIQUE

Field types:

- `invoice` — FK to Invoice, on_delete=CASCADE, required
- `resource_type` — CharField(max_length=50), required
- `resource_id` — PositiveIntegerField, required
- `date` — DateField, required
- `daily_cost` — DecimalField(max_digits=14, decimal_places=10), required (10 decimal places)
- `currency` — CharField(max_length=3, default="NOK")
- `autofilled` — BooleanField(default=False). `true` when the usage was autofilled (carry-forward); `false` when from a real snapshot. This is a dedicated database column to support direct filtering and aggregation queries (e.g., "all autofilled daily costs for this invoice", "count of autofilled days per resource"). `autofilled` is also kept in `metadata` JSON for audit self-containment, but the column is the queryable source of truth.
- `metadata` — JSONField(default=dict)

Recommended partial index: `CREATE INDEX ... ON InvoiceDailyCost (invoice_id) WHERE autofilled = true` — optimizes queries filtering for autofilled rows.

Metadata — required fields (must always be present for audit reproducibility):

- `normalized_usage` — object keyed by pricing dimension. Contains the usage value(s) after unit conversion used in the billing calculation.
- `resolved_prices` — object keyed by pricing dimension. Each entry contains `price_per_unit_year`, `currency`, `discount_applied`.
- `dimension_costs` — object keyed by pricing dimension. Each value is the per-dimension daily cost as a Decimal string.
- `autofilled` — boolean (mirrors the `autofilled` column for audit self-containment; the column is the queryable source of truth)

**Note on example precision:** The `dimension_costs` examples below are simplified for readability. Actual values use full Decimal precision matching `InvoiceDailyCost.daily_cost` (10 decimal places), e.g., `"131.5068493150"` rather than `"131.51"`.

StorageHotel metadata example:

```json
{
  "normalized_usage": {"quota_tb": "120"},
  "resolved_prices": {
    "quota_tb": {"price_per_unit_year": "400", "currency": "NOK", "discount_applied": true}
  },
  "dimension_costs": {"quota_tb": "131.51"},
  "autofilled": false
}
```

VirtualMachine metadata example:

```json
{
  "normalized_usage": {"cpu_count": "8", "ram_gb": "32", "disk_gb": "500"},
  "resolved_prices": {
    "cpu_count": {"price_per_unit_year": "300", "currency": "NOK", "discount_applied": false},
    "ram_gb": {"price_per_unit_year": "40", "currency": "NOK", "discount_applied": false},
    "disk_gb": {"price_per_unit_year": "2", "currency": "NOK", "discount_applied": false}
  },
  "dimension_costs": {"cpu_count": "6.58", "ram_gb": "3.51", "disk_gb": "2.74"},
  "autofilled": false
}
```

Metadata — optional fields:

- `source_snapshot_date` — when autofilled, the date of the original snapshot carried forward
- `missing_data_flags` — additional diagnostic information
- `resource_snapshot` — snapshot of the raw resource data at billing time (optional at this level; required at InvoiceLine level)

**`missing_data_summary` shape** (present in `Invoice.metadata` when `incomplete=true`):

```json
{
  "storage_hotel": {
    "101": {"missing_days": ["2026-01-05", "2026-01-06"], "count": 2}
  },
  "virtual_machine": {
    "205": {"missing_days": ["2026-01-10"], "count": 1}
  }
}
```

Keys are `resource_type` values; nested keys are string representations of `resource_id`.

---

Note: InvoiceDailyCost does not have a FK to InvoiceLine. The relationship is resolved through tuple matching on `(invoice, resource_type, resource_id)`. This is intentional — it avoids FK management complexity during draft replacement.
