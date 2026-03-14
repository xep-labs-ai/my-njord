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
- `contact_point` (CharField)
- `contact_email` (EmailField)
- `contact_telephone_number` (CharField)
- `customer_number` (CharField)
- `make_invoice` (BooleanField, default=True) — controls whether invoice generation runs for this account. Resources belonging to an account where `make_invoice = False` are excluded from all invoice generation runs silently.
- `internal_customer` (BooleanField)

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

- `price_list` (FK to PriceList, required) — each BillingAccount must use exactly one PriceList
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

`active_from` (date, required): the first day the resource is billable.

`active_to` (date, nullable): the last day the resource is billable. `null` means open-ended (no end date).

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
selected_resource_ids
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

- `pricing_dimension` (CharField) — the billing dimension for this daily cost. For StorageHotel rows: `quota_tb`. For VirtualMachine rows: `cpu_count`, `ram_gb`, or `disk_gb`.

Metadata may include:
```
normalized_usage
resolved_prices
dimension_costs
autofilled
source_snapshot_date
missing_data_flags
```
