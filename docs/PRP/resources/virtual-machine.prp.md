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

## Unit Conversion Rules

VirtualMachine daily usage is captured and normalized for billing:

- `cpu_count` → `cpu_count`: 1:1, no conversion
- `ram_mb` → `ram_gb`: divide by 1024 (binary). Example: `ram_gb = Decimal(ram_mb) / Decimal("1024")`
- `disks_total_gb` → `disk_gb`: 1:1, no conversion

---

## Billing Dimensions

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

## Invoice Expectations

For VirtualMachine invoices, `InvoiceLine.metadata` has this standard structure:

```json
{
  "billing_dimensions": ["cpu_count", "ram_gb", "disk_gb"],
  "total_quantity_by_dimension": {
    "cpu_count_days": "248",
    "ram_gb_days": "992",
    "disk_gb_days": "15500"
  },
  "provisioner": "VCENTER"
}
```

Note:

- `cpu_count_days` = CPU count × days
- `ram_gb_days` = RAM in GB × days (converted from MB before aggregation)
- `disk_gb_days` = disk GB × days

`InvoiceDailyCost.metadata` required fields (same as all resources):

- `normalized_usage` — normalized daily usage value for this pricing dimension
- `resolved_prices` — the price(s) per unit applied on that day
- `autofilled` — boolean, whether usage was autofilled

Optional fields (VM-specific):

- `cpu_count` — normalized cpu_count value used
- `ram_gb` — normalized ram_gb value used (converted from ram_mb)
- `disk_gb` — normalized disk_gb value used
- `dimension_costs` — per-dimension cost breakdown
- `source_snapshot_date` — when autofilled, the date of the original snapshot

---

## API endpoints

```text
POST   /api/v1/virtual-machines/
GET    /api/v1/virtual-machines/
GET    /api/v1/virtual-machines/{id}/
PATCH  /api/v1/virtual-machines/{id}/
POST   /api/v1/virtual-machines/{id}/usage
```
