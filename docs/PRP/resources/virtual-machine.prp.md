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

**v1 uniqueness note:** There is no natural-key uniqueness constraint in v1 beyond the primary key. Callers and ingestion workflows are responsible for avoiding duplicate VM creation. Duplicate VirtualMachine rows, if created, are treated as separate billable resources.

---

## Daily snapshot model

### VirtualMachineDailyUsage

All three daily usage fields are **required and non-nullable** in v1. Partial snapshots are not allowed. Autofill always carries forward a complete state.

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

- `cpu_count` → `cpu_count`: 1:1, no conversion. Note: `cpu_count` is stored as `PositiveIntegerField` but is coerced to `Decimal` during billing normalization via `Decimal(str(cpu_count))`. Metadata shows it as a Decimal string (e.g., `"cpu_count": "8"`).
- `ram_mb` → `ram_gb`: divide by 1024 (binary). Example: `ram_gb = Decimal(ram_mb) / Decimal("1024")`
- `disks_total_gb` → `disk_gb`: 1:1, no conversion

Note: "GB" in VirtualMachine context means binary gigabytes (GiB = 1024 MB), consistent with hypervisor conventions.

---

## Billing Dimensions

VirtualMachine uses per-dimension billing. Three dimensions are billable:

```text
cpu_count
ram_gb
disk_gb
```

There is **one `InvoiceDailyCost` row per resource per day**. Per-dimension usage, prices, and costs are stored in the row's `metadata` under `normalized_usage`, `resolved_prices`, and `dimension_costs`. Each dimension has independent pricing and discounts resolved via `ResourcePrice`.

---

## Manager Availability

VirtualMachine inherits the `billing_objects` manager from ResourceModel. This manager includes soft-deleted resources, which is necessary for the billing engine to bill historical periods.

---

## Soft-Delete Invariants

- If `deleted_at` is set, `status` must be `RETIRED`
- If `deleted_at` is set, `active_to` must be set
- `active_to` must be on or before the calendar date of `deleted_at`
- Default querysets exclude soft-deleted resources; use `billing_objects` manager for billing operations
- Billability for historical days is resolved from `active_from`/`active_to`, not from `deleted_at` alone

---

## Autofill Rule

When autofill is needed for a missing day on a VirtualMachine, all three billing dimensions (`cpu_count`, `ram_mb`, `disks_total_gb`) are carried forward together from the last known complete `VirtualMachineDailyUsage` row. Individual dimensions are never autofilled independently. This is an atomic carry-forward operation.

---

## Billing notes

The daily usage row is a captured billing snapshot, not necessarily real runtime utilization.

---

## Canonical `resource_snapshot` Schema

The `resource_snapshot` in `InvoiceLine.metadata` for VirtualMachine must contain:

```json
{
  "id": "<int>",
  "name": "<str>",
  "provisioner": "<str>"
}
```

This snapshot is frozen at invoice generation time and must be present for all VirtualMachine InvoiceLines.

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
  "provisioner": "VCENTER",
  "resource_snapshot": {
    "id": 205,
    "name": "vm-prod-001",
    "provisioner": "VCENTER"
  }
}
```

Required fields in `InvoiceLine.metadata`:

- `resource_snapshot` — frozen resource snapshot for audit reproducibility
- `provisioner` — records the provisioner at billing time

Note:

- `cpu_count_days` = CPU count × days
- `ram_gb_days` = RAM in GB × days (converted from MB before aggregation)
- `disk_gb_days` = disk GB × days

---

`InvoiceDailyCost.metadata` required fields:

- `normalized_usage` — object keyed by pricing dimension: `{"cpu_count": "<value>", "ram_gb": "<value>", "disk_gb": "<value>"}`
- `resolved_prices` — object keyed by pricing dimension, each entry contains `price_per_unit_year`, `currency`, `discount_applied`
- `dimension_costs` — object keyed by pricing dimension, each value is the per-dimension daily cost as a Decimal string
- `autofilled` — boolean, whether usage was autofilled

VirtualMachine `InvoiceDailyCost.metadata` example:

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

Optional fields:

- `source_snapshot_date` — when autofilled, the date of the original snapshot
- `resource_snapshot` — optional at daily-cost level

---

## API endpoints

```text
POST   /api/v1/virtual-machines/
GET    /api/v1/virtual-machines/
GET    /api/v1/virtual-machines/{id}/
PATCH  /api/v1/virtual-machines/{id}/
POST   /api/v1/virtual-machines/{id}/usage
```
