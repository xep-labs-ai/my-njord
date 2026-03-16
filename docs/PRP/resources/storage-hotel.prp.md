# StorageHotel

## Purpose

Represents a filesystem resource billed from daily quota data.

---

## Model

### StorageHotel (inherits ResourceModel)

Fields:

```text
id
billing_account
name
filesystem_identifier
quota_unit
status
active_from
active_to
created_at
updated_at
deleted_at
```

`quota_unit` values:

```text
KB
KIB
```

Constraint:

```text
filesystem_identifier UNIQUE
```

---

## Daily snapshot model

### StorageHotelDailyQuota

Fields:

```text
id
storage_hotel
date
quota_raw
created_at
```

Constraint:

```text
(storage_hotel_id, date) UNIQUE
```

`quota_raw` is stored in the unit defined by `quota_unit`.

---

## Ingestion event model

### QuotaIngestionEvent

Fields:

```text
id
storage_hotel
date
raw_payload
normalized_quota_raw
request_id
created_at
```

---

## Billing unit

Normalized billing unit:

```text
TB
```

Conversion is based on `quota_unit`.

## Unit Conversion Rules

Billing unit is decimal **TB** (10^12 bytes).

Conversion formulas:

- `KB → TB`: divide by 1,000,000,000 (`Decimal("1e-9")`)
- `KIB → TB`: multiply by 1024, then divide by 1,000,000,000,000 (i.e., `Decimal("1024") / Decimal("1e12")`)

---

## Manager Availability

StorageHotel inherits the `billing_objects` manager from ResourceModel. This manager includes soft-deleted resources, which is necessary for the billing engine to bill historical periods.

## Soft-Delete Invariants

- If `deleted_at` is set, `status` must be `RETIRED`
- If `deleted_at` is set, `active_to` must be set
- `active_to` must be on or before the calendar date of `deleted_at`
- Default querysets exclude soft-deleted resources; use `billing_objects` manager for billing operations
- Billability for historical days is resolved from `active_from`/`active_to`, not from `deleted_at` alone

---

## Billing notes

Daily billing uses quota converted to TB, then applies the effective yearly price for that day.

The StorageHotel `pricing_dimension` on `ResourcePrice` is always `"quota_tb"` in v1.

---

## Canonical `resource_snapshot` Schema

The `resource_snapshot` in `InvoiceLine.metadata` for StorageHotel must contain:

```json
{
  "id": "<int>",
  "name": "<str>",
  "filesystem_identifier": "<str>",
  "quota_unit": "<str>"
}
```

This snapshot is frozen at invoice generation time and must be present for all StorageHotel InvoiceLines.

---

## Invoice expectations

`InvoiceLine.metadata` standard structure:

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

Note: `resource_snapshot` uses the canonical schema defined in the "Canonical `resource_snapshot` Schema" section above and is required for all StorageHotel InvoiceLines.

Note: `quota_unit` is a required field in InvoiceLine metadata for StorageHotel to enable audit verification of unit conversion.

`InvoiceDailyCost.metadata` uses the nested keyed shape. `normalized_usage` uses `{"quota_tb": "<value>"}` -- not a scalar.

Required fields: `normalized_usage`, `resolved_prices`, `dimension_costs`, `autofilled`.

StorageHotel `InvoiceDailyCost.metadata` example:

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

Optional fields: `source_snapshot_date`, `missing_data_flags`, `resource_snapshot`.

---

## API endpoints

```text
POST   /api/v1/storage-hotels/
GET    /api/v1/storage-hotels/
GET    /api/v1/storage-hotels/{id}/
PATCH  /api/v1/storage-hotels/{id}/
POST   /api/v1/storage-hotels/{id}/quota
```
