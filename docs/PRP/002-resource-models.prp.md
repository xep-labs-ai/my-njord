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

Fields:

```
billing_account
name
status
```

Status lifecycle:

```
UNASSIGNED
ACTIVE
RETIRED
```

---

# StorageHotel

Represents a **filesystem resource**.

Fields:

```
billing_account
filesystem_identifier
quota_unit
status
created_at
updated_at
deleted_at
```

Quota units:

```
KB
KIB
```

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
created_at
updated_at
deleted_at
```

Provisioners:

```
VCENTER
```

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
total_billed_amount
unit_price_snapshot
total_cost
currency
metadata
```

Metadata may include:
```
billing_dimensions
total_quantity_by_dimension
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
