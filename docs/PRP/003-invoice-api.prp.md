# Invoice API Specification

## Purpose

This document defines the HTTP contract for invoice generation, retrieval, and finalization.

It specifies endpoint paths, request/response shapes, selection inputs, validation rules, and lifecycle transitions for invoices.

This document complements `001-billing-engine.prp.md`, which defines the billing calculation rules. The billing engine PRP is the source of truth for billing logic. This API PRP is the source of truth for the HTTP interface.

**Global rule:** `selection_fingerprint` is excluded from all public API responses in v1. It is an internal implementation detail used for duplicate prevention.

---

## Endpoints (v1)

All invoice endpoints live under `/api/v1/invoices/`.

### POST /api/v1/invoices/generate

Generate a draft invoice.

**Request body:**

```json
{
  "billing_account": 1,
  "period_start": "2026-01-01",
  "period_end": "2026-01-31",
  "selection_scope": "all_resources|resource_types|explicit_resources",
  "selected_resource_types": ["storage_hotel", "virtual_machine"],
  "explicit_resources": [
    {"resource_type": "storage_hotel", "resource_id": 101},
    {"resource_type": "virtual_machine", "resource_id": 205}
  ],
  "autofill_missing_days": true,
  "force": false
}
```

**Response:**

- Status 201: Invoice created
- Status 400: Validation error (missing data, invalid selection, etc.)
- Status 409: Duplicate draft or finalized invoice exists (unless force=true)

The generate endpoint returns the full invoice including lines, using the same serializer shape as the detail endpoint.

**Response body:**

```json
{
  "id": "<invoice id>",
  "invoice_number": null,
  "billing_account": "<id>",
  "period_start": "2026-01-01",
  "period_end": "2026-01-31",
  "status": "draft",
  "total_amount": "1500.50",
  "currency": "NOK",
  "selection_scope": "resource_types",
  "created_at": "2026-01-15T10:30:00Z",
  "updated_at": "2026-01-15T10:30:00Z",
  "finalized_at": null,
  "incomplete": false,
  "metadata": {
    "selected_resource_types": ["storage_hotel"],
    "autofill_missing_days": true,
    "force": false,
    "provisional": false,
    "missing_data_summary": null
  },
  "lines": [
    {
      "id": "<id>",
      "resource_type": "storage_hotel",
      "resource_id": 101,
      "description": "StorageHotel #101",
      "total_cost": "1500.5000000000",
      "currency": "NOK",
      "metadata": {
        "billing_dimensions": ["quota_tb"],
        "total_quantity_by_dimension": {
          "quota_tb_days": "31.5"
        },
        "quota_unit": "KIB",
        "resource_snapshot": {
          "id": 101,
          "name": "storage-primary",
          "filesystem_identifier": "storage-001",
          "quota_unit": "KIB"
        }
      }
    }
  ]
}
```

### GET /api/v1/invoices/

List invoices.

This endpoint returns a **reduced summary serializer** intentionally. Full metadata is available on the detail endpoint.

**Query parameters:**

- `billing_account` (optional): filter by billing account
- `status` (optional): filter by status (draft, finalized)
- `period_start` (optional): filter by period start date
- `period_end` (optional): filter by period end date
- `incomplete` (optional, boolean): filter by incomplete flag
- `selection_scope` (optional): filter by selection scope (matches the top-level `selection_scope` column on the Invoice model)

**Response:**

- Status 200: List of invoices

**Fields included in list response:**

`id`, `invoice_number`, `billing_account`, `period_start`, `period_end`, `currency`, `status`, `total_amount`, `selection_scope`, `created_at`, `updated_at`, `finalized_at`, `incomplete`

**Fields intentionally excluded from list response:**

`metadata`, `selection_fingerprint`, `provisional`

Note: `provisional` is intentionally excluded from the list response in v1. It is a detail-level concern and is available in the detail and generate responses only (inside `metadata`).

**Response body:**

```json
{
  "count": 42,
  "next": "http://api/v1/invoices/?page=2",
  "previous": null,
  "results": [
    {
      "id": "<id>",
      "invoice_number": null,
      "billing_account": 1,
      "period_start": "2026-01-01",
      "period_end": "2026-01-31",
      "status": "draft",
      "total_amount": null,
      "currency": "NOK",
      "selection_scope": "resource_types",
      "created_at": "2026-01-15T10:30:00Z",
      "updated_at": "2026-01-15T10:30:00Z",
      "finalized_at": null,
      "incomplete": false
    }
  ]
}
```

### GET /api/v1/invoices/{id}/

Retrieve a single invoice with lines.

**Response:**

- Status 200: Invoice with lines
- Status 404: Invoice not found

**Response body:**

```json
{
  "id": "<id>",
  "invoice_number": null,
  "billing_account": "<id>",
  "period_start": "2026-01-01",
  "period_end": "2026-01-31",
  "status": "draft",
  "total_amount": "1500.50",
  "currency": "NOK",
  "selection_scope": "resource_types",
  "created_at": "2026-01-15T10:30:00Z",
  "updated_at": "2026-01-15T10:30:00Z",
  "finalized_at": null,
  "incomplete": false,
  "metadata": {
    "selected_resource_types": ["storage_hotel"],
    "autofill_missing_days": true,
    "force": false,
    "provisional": false
  },
  "lines": [
    {
      "id": "<id>",
      "resource_type": "storage_hotel",
      "resource_id": 101,
      "description": "StorageHotel #101",
      "total_cost": "1500.5000000000",
      "currency": "NOK",
      "metadata": {
        "billing_dimensions": ["quota_tb"],
        "total_quantity_by_dimension": {
          "quota_tb_days": "31.5"
        },
        "quota_unit": "KIB",
        "resource_snapshot": {
          "id": 101,
          "name": "storage-primary",
          "filesystem_identifier": "storage-001",
          "quota_unit": "KIB"
        }
      }
    }
  ]
}
```

### POST /api/v1/invoices/{id}/finalize

Transition a draft invoice to finalized.

Once finalized, the invoice becomes immutable. Returns the full finalized Invoice object using the same serializer shape as the detail endpoint.

**Request body:**

```json
{}
```

**Response:**

- Status 200: Invoice finalized
- Status 404: Invoice not found
- Status 409: Invoice is already finalized

**Response body:**

```json
{
  "id": "<id>",
  "invoice_number": "INV-2026-01-00001",
  "billing_account": "<id>",
  "period_start": "2026-01-01",
  "period_end": "2026-01-31",
  "status": "finalized",
  "total_amount": "1500.50",
  "currency": "NOK",
  "selection_scope": "resource_types",
  "created_at": "2026-01-15T10:30:00Z",
  "updated_at": "2026-01-15T11:00:00Z",
  "finalized_at": "2026-01-15T11:00:00Z",
  "incomplete": false,
  "metadata": {
    "selected_resource_types": ["storage_hotel"],
    "autofill_missing_days": true,
    "force": false,
    "provisional": false
  },
  "lines": [
    {
      "id": "<id>",
      "resource_type": "storage_hotel",
      "resource_id": 101,
      "description": "StorageHotel #101",
      "total_cost": "1500.5000000000",
      "currency": "NOK",
      "metadata": {
        "billing_dimensions": ["quota_tb"],
        "total_quantity_by_dimension": {
          "quota_tb_days": "31.5"
        },
        "quota_unit": "KIB",
        "resource_snapshot": {
          "id": 101,
          "name": "storage-primary",
          "filesystem_identifier": "storage-001",
          "quota_unit": "KIB"
        }
      }
    }
  ]
}
```

---

## BillingAccount Reference

In the request body, `billing_account` is the integer primary key of `BillingAccount`.

Business fields such as `name`, `customer_number`, or accounting identifiers must not be used as the invoice-generation identifier.

---

## Selection Input Contract

Explicit resource selection uses `(resource_type, resource_id)` pairs to avoid ID ambiguity:

```json
explicit_resources = [
  {"resource_type": "storage_hotel", "resource_id": 101},
  {"resource_type": "virtual_machine", "resource_id": 205}
]
```

This format is consistent with the internal `resource_type + resource_id` pattern used by `InvoiceLine` and `InvoiceDailyCost`.

---

## Validation Failure Cases

Invoice generation uses three error status codes:

### 400 Bad Request — request validation failures

Returned when the request itself is malformed or contains invalid parameters:

- missing required JSON field
- invalid date format
- `period_start` is after `period_end` (note: `period_start == period_end` is valid and produces exactly one daily evaluation)
- `period_end > today` and `autofill_missing_days` is not `true`
- requested resource types are unknown or unsupported
- the selection is empty or ambiguous
- the same resource is selected more than once
- selected resources do not belong to the provided billing account

### 409 Conflict — duplicate or state conflicts

Returned when the request conflicts with existing state:

- `duplicate_draft_invoice` — a matching draft invoice already exists (unless `force=true`)
- `duplicate_finalized_invoice` — a matching finalized invoice already exists (immutability rule)
- `invoice_already_finalized` — attempt to operate on a finalized invoice (used by finalize endpoint)

### 422 Unprocessable Entity — billing domain failures

Returned when the request is valid but the billing engine cannot complete the operation:

- `no_billable_resources` — selection produced no billable resources for the period
- `missing_snapshot` — resource has a missing day with no prior state and `force=false`
- `missing_pricing` — no valid `ResourcePrice` found for a resource on a billed day (always fatal, even with `force=true`)
- `billing_account_not_billable` — `make_invoice=false` on the billing account

### Error Response Body Examples

All error responses use the structured format defined in `.claude/docs/API.md`.

**409 Conflict example (`duplicate_draft_invoice`):**

```json
{
  "code": "duplicate_draft_invoice",
  "message": "A draft invoice already exists for the same billing account, billing period, and billing selection.",
  "details": {
    "billing_account": 123,
    "period_start": "2026-01-01",
    "period_end": "2026-01-31",
    "selection_scope": "all_resources"
  }
}
```

**422 Unprocessable Entity example (`missing_snapshot`):**

```json
{
  "code": "missing_snapshot",
  "message": "Invoice generation failed because one or more required billing snapshots were missing.",
  "details": {
    "resource_type": "storage_hotel",
    "resource_id": 101,
    "missing_dates": ["2026-01-16", "2026-01-17", "2026-01-18"]
  }
}
```

---

## Draft vs. Finalized Behavior

### Draft invoice

- may be retrieved and inspected
- may be replaced by calling POST /generate with `force=true` and matching parameters
- can be finalized via POST /{id}/finalize

### Finalized invoice

- immutable — no modifications
- cannot be finalized again
- snapshot rows are fixed
- must remain reproducible even if source pricing or usage data later changes

---

## Duplicate Prevention Behavior

When an invoice generation request is made:

1. Check for a matching finalized invoice using `(billing_account, period_start, period_end, selection_scope, selection_fingerprint)`. `selection_fingerprint` is a deterministic hash derived from `selected_resource_types` and `explicit_resources` (see `001-billing-engine.prp.md` for the fingerprint specification).
   - If found, return 409 Conflict. Finalized invoices are immutable.

2. Check for a matching draft invoice using the same `(billing_account, period_start, period_end, selection_scope, selection_fingerprint)` tuple.
   - If found and `force=false`: return 409 Conflict.
   - If found and `force=true`: replace the draft invoice atomically.

---

## Force=true Regeneration Rules

When `force=true` is passed:

- If a matching draft invoice exists, it is deleted with all its children (InvoiceLines, InvoiceDailyCosts) in the same transaction, and a new draft is created. The replacement draft has `invoice_number = null` (consistent with all draft invoices).
- If a matching finalized invoice exists, invoice generation fails (immutability)
- Missing data that would normally fail is allowed, according to the billing engine rules
- The invoice metadata will reflect that `force` was used
- If `force=true` and `autofill_missing_days=false`, missing days are billed at zero

---

## Status Transitions

Valid invoice status transitions:

```
DRAFT → FINALIZED
```

No reverse transitions are allowed. A finalized invoice cannot return to draft.

---

## No DELETE Endpoint for Draft Invoices (v1 Intentional Design)

Draft invoices cannot be deleted via the API in v1. They are replaced through regeneration with `force=true`.

Finalized invoices are immutable and cannot be modified or deleted.

This intentional constraint ensures that invoice metadata and selection fingerprints remain auditable.

---

## Optional Future Endpoints (v2+)

These endpoints are noted for future development but are not part of v1:

```
GET    /api/v1/invoices/{id}/lines          — retrieve invoice lines in detail
GET    /api/v1/invoices/{id}/daily-costs    — retrieve daily-cost breakdown
POST   /api/v1/invoices/{id}/recalculate    — recalculate a draft (before finalization)
DELETE /api/v1/invoices/{id}                — delete a draft invoice
```

For v1, retrieval of lines is included in the main GET /{id}/ response.

### POST /api/v1/invoices/{id}/recalculate (v2 stub)

Draft-only operation. Re-executes billing for an existing draft invoice using the invoice's stored billing context and selection snapshot. Reuses `billing_account`, `period_start`, `period_end`, `selection_scope`, and stored selection data. Deletes and rebuilds all existing `InvoiceLine` and `InvoiceDailyCost` rows. Recomputes invoice totals. Preserves the same Invoice record and draft status. `force` and `autofill_missing_days` flags may be re-specified. Finalized invoices cannot be recalculated. Recalculation does not change invoice identity or selection scope.
