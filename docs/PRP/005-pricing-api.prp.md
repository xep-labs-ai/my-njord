# Pricing API Specification

## Purpose

This document defines the HTTP contract for managing `PriceList`, `ResourcePrice`, and `BillingAccount` entities.

These are configuration entities that control billing behavior. This document is the source of truth for their HTTP interface. Domain model definitions live in `002-resource-models.prp.md`. Billing calculation rules live in `001-billing-engine.prp.md`.

---

## Common Rules

All endpoints are under `/api/v1/`.

Request and response bodies use JSON.

All list endpoints use standard DRF pagination (`count`, `next`, `previous`, `results`).

---

## BillingAccount Endpoints

### POST /api/v1/billing-accounts/

Create a new BillingAccount.

**Required fields:**

- `name` (unique)
- `price_list` (integer PK of PriceList)

**Optional fields:**

- `contact_point`
- `contact_email`
- `contact_telephone_number`
- `customer_number`
- `make_invoice` (default: true)
- `internal_customer` (default: true)
- `usit_contact_point`
- `main_agreement_id`
- `main_agreement_description`
- `usit_accounting_place`
- `usit_sub_project`
- `ephorte`
- `uio_unit`

**Request example:**

```json
{
  "name": "Department of Physics",
  "price_list": 1,
  "contact_email": "physics-it@uio.no",
  "customer_number": "UIO-1234",
  "make_invoice": true,
  "internal_customer": true,
  "uio_unit": "PHYS"
}
```

**Response (201):**

```json
{
  "id": 1,
  "name": "Department of Physics",
  "price_list": 1,
  "contact_point": null,
  "contact_email": "physics-it@uio.no",
  "contact_telephone_number": null,
  "customer_number": "UIO-1234",
  "make_invoice": true,
  "internal_customer": true,
  "usit_contact_point": null,
  "main_agreement_id": null,
  "main_agreement_description": null,
  "usit_accounting_place": null,
  "usit_sub_project": null,
  "ephorte": null,
  "uio_unit": "PHYS",
  "created_at": "2026-01-01T00:00:00Z",
  "updated_at": "2026-01-01T00:00:00Z"
}
```

**Validation:**

- `name` must be unique
- `price_list` must reference an existing PriceList
- Returns 400 if required fields are missing or validation fails

---

### GET /api/v1/billing-accounts/

List all billing accounts.

**Query parameters:**

- `make_invoice` (optional): filter by true/false
- `internal_customer` (optional): filter by true/false

**Response (200):** Paginated list.

---

### GET /api/v1/billing-accounts/{id}/

Retrieve a single billing account.

**Response:**

- 200: BillingAccount object
- 404: Not found

---

### PATCH /api/v1/billing-accounts/{id}/

Partial update. All fields are patchable except `id`, `created_at`, `updated_at`.

**v1 limitation — price list reassignment:**

> If `BillingAccount.price_list` is changed after historical usage has been captured, future invoice generation for uninvoiced periods will use the current `price_list`. Previously generated draft invoices for affected periods should be regenerated before finalization. Already finalized invoices remain immutable. This should be revisited if historical price-list assignment tracking becomes necessary.

**Response:**

- 200: Updated BillingAccount
- 400: Validation error
- 404: Not found

---

## PriceList Endpoints

### POST /api/v1/price-lists/

Create a new PriceList.

**Required fields:**

- `name` (unique)

**Request example:**

```json
{
  "name": "Standard 2026"
}
```

**Response (201):**

```json
{
  "id": 1,
  "name": "Standard 2026",
  "created_at": "2026-01-01T00:00:00Z",
  "updated_at": "2026-01-01T00:00:00Z"
}
```

**Validation:**

- `name` must be unique
- Returns 400 if validation fails

---

### GET /api/v1/price-lists/

List all price lists.

**Response (200):** Paginated list.

---

### GET /api/v1/price-lists/{id}/

Retrieve a single price list with its resource prices.

**Response:**

- 200: PriceList object
- 404: Not found

---

### PATCH /api/v1/price-lists/{id}/

Partial update. Only `name` is patchable.

**Response:**

- 200: Updated PriceList
- 400: Validation error
- 404: Not found

---

## ResourcePrice Endpoints

ResourcePrice is **immutable** — rows are never updated after creation. To change a price, set `effective_to` on the existing row (via a dedicated endpoint) and create a new row.

Nested under PriceList to make ownership explicit.

### POST /api/v1/price-lists/{price_list_id}/resource-prices/

Create a new ResourcePrice row.

**Required fields:**

- `resource_type` — snake_case, e.g. `"storage_hotel"`, `"virtual_machine"`
- `pricing_dimension` — e.g. `"quota_tb"`, `"cpu_count"`, `"ram_gb"`, `"disk_gb"`
- `price_per_unit_year` — Decimal
- `effective_from` — date

**Optional fields:**

- `price_currency` (default: `"NOK"`)
- `discount_price_per_unit_year` — Decimal, null = no discount
- `discount_threshold_quantity` — Decimal, null = discount does not apply
- `effective_to` — date, null = open-ended

**Request example:**

```json
{
  "resource_type": "storage_hotel",
  "pricing_dimension": "quota_tb",
  "price_per_unit_year": "500.0000",
  "discount_price_per_unit_year": "400.0000",
  "discount_threshold_quantity": "10.0000",
  "effective_from": "2026-01-01",
  "effective_to": null
}
```

**Response (201):**

```json
{
  "id": 1,
  "price_list": 1,
  "resource_type": "storage_hotel",
  "pricing_dimension": "quota_tb",
  "price_per_unit_year": "500.0000",
  "price_currency": "NOK",
  "discount_price_per_unit_year": "400.0000",
  "discount_threshold_quantity": "10.0000",
  "effective_from": "2026-01-01",
  "effective_to": null,
  "created_at": "2026-01-01T00:00:00Z"
}
```

**Validation:**

- No two ResourcePrice rows for the same `(price_list, resource_type, pricing_dimension)` may have overlapping effective date ranges — validated at the service layer (for clear API error messages) and enforced by a PostgreSQL `daterange` exclusion constraint (authoritative enforcement)
- `effective_to` must be >= `effective_from` if set (a single-day price range is valid)
- `price_per_unit_year` must be positive
- `discount_price_per_unit_year` must be positive if set
- **Cross-field validation:** `discount_price_per_unit_year` and `discount_threshold_quantity` must both be set or both be null. Storing only one of the two is a validation error (400).
- Returns 400 on validation failure, 409 with error code `price_range_overlap` on date range overlap

---

### GET /api/v1/price-lists/{price_list_id}/resource-prices/

List all ResourcePrice rows for a PriceList.

**Query parameters:**

- `resource_type` (optional): filter
- `pricing_dimension` (optional): filter

**Response (200):** Paginated list.

---

### GET /api/v1/price-lists/{price_list_id}/resource-prices/{id}/

Retrieve a single ResourcePrice row.

**Response:**

- 200: ResourcePrice object
- 404: Not found

---

### PATCH /api/v1/price-lists/{price_list_id}/resource-prices/{id}/set-effective-to/

Sets `effective_to` on an **open-ended** ResourcePrice row (one with `effective_to = null`). This is the only mutation allowed on a ResourcePrice row after creation.

This endpoint uses a dedicated DRF `@action` named `set_effective_to` to constrain this specific lifecycle operation.

Time-bounded rows (those with `effective_to` already set) are immutable — to correct a time-bounded row, create a new row with the correct dates instead.

**Request body:**

```json
{
  "effective_to": "2026-12-31"
}
```

**Validation:**

- `effective_to` must be >= `effective_from` (a single-day price range is valid)
- `effective_to` must not overlap with any existing row for the same `(price_list, resource_type, pricing_dimension)`
- This endpoint only works on open-ended rows (where `effective_to` is currently null). If the row already has an `effective_to` set, return 409 with error code `price_row_already_closed`.

**Response:**

- 200: Updated ResourcePrice
- 400: Validation error
- 404: Not found
- 409: Row is not open-ended (`price_row_already_closed`) or overlap detected

---

## No DELETE Endpoints

No delete endpoints are provided for any pricing entity in v1.

- `PriceList` deletion is not supported — a PriceList in use by a BillingAccount must not be deleted.
- `ResourcePrice` rows are immutable records and must not be deleted (audit trail).
- `BillingAccount` deletion is not supported in v1 — deactivation via `make_invoice = False` is the preferred approach.
