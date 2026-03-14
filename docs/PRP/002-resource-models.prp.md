# Resource Models

Defines all **billable resource models**.

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
total_cost
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
billing_unit
total_cost
currency
metadata
```

Metadata must include `total_quantity_by_dimension` storing aggregate billed quantities per dimension.

Example for VirtualMachine:
```
metadata.total_quantity_by_dimension = {
  "cpu_count_days": "248",
  "ram_gb_days": "992",
  "disk_gb_days": "15500"
}
```

For single-dimension resources like StorageHotel, a simpler key is used (e.g., `quota_tb_days`).

Metadata may also include:
```
billing_dimensions
price_summary
provisioner
quota_unit
```

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
daily_cost
currency
metadata
created_at
```

Constraint:
(invoice_id, resource_type, resource_id, date) UNIQUE

Metadata may include:
```
normalized_usage
resolved_prices
dimension_costs
autofilled
source_snapshot_date
missing_data_flags
```
