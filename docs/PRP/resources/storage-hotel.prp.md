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

## Soft-Delete Invariants

- If `deleted_at` is set, `status` must be `RETIRED`
- If `deleted_at` is set, `active_to` must be set
- `active_to` must be on or before the calendar date of `deleted_at`
- Default querysets exclude soft-deleted resources
- Billability for historical days is resolved from `active_from`/`active_to`, not from `deleted_at` alone

---

## Billing notes

Daily billing uses quota converted to TB, then applies the effective yearly price for that day.

---

## Invoice expectations

`InvoiceLine.metadata` may include:

```text
total_quota_days_tb
quota_unit
filesystem_identifier
```

`InvoiceDailyCost.metadata` may include:

```text
quota_raw
quota_tb
quota_unit
```

---

## API endpoints

```text
POST   /api/v1/storage-hotels/
GET    /api/v1/storage-hotels/
GET    /api/v1/storage-hotels/{id}/
PATCH  /api/v1/storage-hotels/{id}/
POST   /api/v1/storage-hotels/{id}/quota
```
