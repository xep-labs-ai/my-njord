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
POST   /storage-hotels/
GET    /storage-hotels/
GET    /storage-hotels/{id}/
PATCH  /storage-hotels/{id}/
POST   /storage-hotels/{id}/quota
```
