# VirtualMachine

## Purpose

Represents a compute resource billed from daily captured VM capacity data.

---

## Model

### VirtualMachine (inherits ResourceModel)

Fields:

```text
id
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

`provisioner` values:

```text
VCENTER
```

---

## Daily snapshot model

### VirtualMachineDailyUsage

Fields:

```text
id
virtual_machine
date
cpu_count
ram_mb
disks_total_gb
created_at
```

Constraint:

```text
(virtual_machine_id, date) UNIQUE
```

---

## Ingestion event model

### VirtualMachineUsageIngestionEvent

Fields:

```text
id
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

## Billing dimensions

VirtualMachine uses per-dimension billing. Three dimensions are billable:

```text
cpu_count
ram_gb
disk_gb
```

Each dimension has its own daily `InvoiceDailyCost` row, allowing independent pricing and discounts per dimension.

---

## Soft-Delete Invariants

- If `deleted_at` is set, `status` must be `RETIRED`
- If `deleted_at` is set, `active_to` must be set
- `active_to` must be on or before the calendar date of `deleted_at`
- Default querysets exclude soft-deleted resources
- Billability for historical days is resolved from `active_from`/`active_to`, not from `deleted_at` alone

---

## Billing notes

The daily usage row is a captured billing snapshot, not necessarily real runtime utilization.

---

## Invoice expectations

`InvoiceLine.metadata` may include:

```text
total_cpu_days
total_ram_mb_days
total_disks_gb_days
provisioner
```

`InvoiceDailyCost.metadata` may include:

```text
cpu_count
ram_mb
disks_total_gb
```

---

## API endpoints

```text
POST   /api/v1/virtual-machines/
GET    /api/v1/virtual-machines/
GET    /api/v1/virtual-machines/{id}/
PATCH  /api/v1/virtual-machines/{id}/
POST   /api/v1/virtual-machines/{id}/usage
```
