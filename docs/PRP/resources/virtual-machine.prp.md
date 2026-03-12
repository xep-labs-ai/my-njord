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

## Billing unit

This resource may require more than one normalized billing measure.

Examples:

```text
cpu_count
ram_mb
disks_total_gb
```

The exact v1 pricing strategy must define whether billing is based on one combined capacity formula or separate dimensions.

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
POST   /virtual-machines/
GET    /virtual-machines/
GET    /virtual-machines/{id}/
PATCH  /virtual-machines/{id}/
POST   /virtual-machines/{id}/usage
```
