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
- `make_invoice` (BooleanField, default=True) — controls whether invoice generation runs for this account. Resources belonging to an account where `make_invoice = False` are excluded from all invoice generation runs. The `POST /api/v1/invoices/generate` endpoint returns 400 if the billing account has `make_invoice = False`.
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
- `price_list` (FK to PriceList, required) — each BillingAccount must use exactly one PriceList
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

Fields:

```
billing_account
name
status
active_from
active_to
```

Field types:

- `billing_account` — FK to BillingAccount, nullable (null = unassigned), on_delete=PROTECT
- `name` — CharField(max_length=255), required, not unique
- `status` — CharField(max_length=20, choices=["UNASSIGNED", "ACTIVE", "RETIRED"], default="UNASSIGNED")
- `active_from` (date, required): the first day the resource is billable.
- `active_to` (date, nullable): the last day the resource is billable. `null` means open-ended (no end date).

Status lifecycle:

```
UNASSIGNED
ACTIVE
RETIRED
```

Per-day billability rule:

A resource is billable for a given day only if:

- `billing_account IS NOT NULL`
- `active_from <= day`
- `active_to IS NULL OR day <= active_to`

If `status == RETIRED`, `active_to` should be set to the final billable day.

---

# StorageHotel

Represents a **filesystem resource**.

Fields:

```
billing_account
filesystem_identifier
quota_unit
status
active_from
active_to
created_at
updated_at
deleted_at
```

Quota units:

```
KB
KIB
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

Fields:

```
billing_account
name
status
provisioner
active_from
active_to
created_at
updated_at
deleted_at
```

Provisioners:

```
VCENTER
```

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
created_at
finalized_at
metadata
```

Metadata may include:
```
selection_scope
selected_resource_types
explicit_resources
force
autofill_missing_days
incomplete
missing_data_summary
```

Uniqueness constraint:

There must be at most one draft invoice per `(billing_account, period_start, period_end, selection_scope, selected_resource_types, explicit_resources)`.

A matching finalized invoice must block regeneration entirely (finalized invoices are immutable).

A matching draft is replaced atomically when `force=true`.

Invoice number assignment:

`invoice_number` is `null` on draft invoices. It is assigned during finalization and is immutable once set.

Field types:

- `invoice_number` — CharField(max_length=20), nullable, unique when set (fits `INV-YYYY-mm-NNNNN`)
- `billing_account` — FK to BillingAccount, required
- `period_start` — DateField, required
- `period_end` — DateField, required
- `currency` — CharField(max_length=3, default="NOK")
- `status` — CharField(max_length=20, choices=["draft", "finalized"], default="draft")
- `total_amount` — DecimalField(max_digits=12, decimal_places=2), nullable — null only before generation runs; set during draft creation and updated on recalculation
- `metadata` — JSONField(default=dict)
- `finalized_at` — DateTimeField, nullable
- `created_at` / `updated_at` — DateTimeField, auto


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

Standard metadata structure for StorageHotel:

```json
{
  "billing_dimensions": ["quota_tb"],
  "total_quantity_by_dimension": {
    "quota_tb_days": "3720"
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
  }
}
```

Metadata may also include:
```
price_summary
provisioner
quota_unit
```

Multi-dimension resources:

For multi-dimension resources (e.g. VirtualMachine), one InvoiceLine is created per resource. The `total_cost` is the sum of all `InvoiceDailyCost` rows for that resource across all dimensions and all days in the billing period.

Field types:

- `invoice` — FK to Invoice, required
- `resource_type` — CharField(max_length=50), required
- `resource_id` — PositiveIntegerField, required
- `description` — CharField(max_length=255), optional (blank=True, null=True)
- `total_cost` — DecimalField(max_digits=14, decimal_places=6), required (full precision, not rounded)
- `currency` — CharField(max_length=3, default="NOK")
- `metadata` — JSONField(default=dict)

---

# InvoiceDailyCost

Daily billing breakdown.

Fields:

```
id
invoice
resource_type
resource_id
date
pricing_dimension
daily_cost
currency
metadata
created_at
```

Constraint:
(invoice_id, resource_type, resource_id, date, pricing_dimension) UNIQUE

Fields:

Field types:

- `invoice` — FK to Invoice, required
- `resource_type` — CharField(max_length=50), required
- `resource_id` — PositiveIntegerField, required
- `pricing_dimension` — CharField(max_length=50), required. For StorageHotel rows: `quota_tb`. For VirtualMachine rows: `cpu_count`, `ram_gb`, or `disk_gb`.
- `date` — DateField, required
- `daily_cost` — DecimalField(max_digits=14, decimal_places=6), required (full precision)
- `currency` — CharField(max_length=3, default="NOK")
- `metadata` — JSONField(default=dict)

Metadata — required fields (must always be present for audit reproducibility):

- `normalized_usage` — the usage value after unit conversion used in the billing calculation
- `resolved_prices` — the price(s) per unit applied on that day (from ResourcePrice)
- `autofilled` — boolean, whether the usage was autofilled (true) or from a real snapshot (false)

Metadata — optional fields:

- `source_snapshot_date` — when autofilled, the date of the original snapshot carried forward
- `dimension_costs` — per-dimension cost breakdown (strongly recommended for multi-dimension resources)
- `missing_data_flags` — additional diagnostic information
- `resource_snapshot` — snapshot of the raw resource data at billing time

Note: InvoiceDailyCost does not have a FK to InvoiceLine. The relationship is resolved through tuple matching on `(invoice, resource_type, resource_id)`. This is intentional — it avoids FK management complexity during draft replacement.
