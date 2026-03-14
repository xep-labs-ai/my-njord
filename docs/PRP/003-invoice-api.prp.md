# Invoice API Specification

## Purpose

This document defines the HTTP contract for invoice generation, retrieval, and finalization.

It specifies endpoint paths, request/response shapes, selection inputs, validation rules, and lifecycle transitions for invoices.

This document complements `001-billing-engine.prp.md`, which defines the billing calculation rules. The billing engine PRP is the source of truth for billing logic. This API PRP is the source of truth for the HTTP interface.

---

## Endpoints (v1)

All invoice endpoints live under `/api/v1/invoices/`.

### POST /api/v1/invoices/generate

Generate a draft invoice.

**Request body:**

```json
{
  "billing_account": "<id or identifier>",
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
  "created_at": "2026-01-15T10:30:00Z",
  "metadata": {
    "selection_scope": "resource_types",
    "selected_resource_types": ["storage_hotel"],
    "autofill_missing_days": true,
    "force": false,
    "incomplete": false,
    "missing_data_summary": null
  }
}
```

### GET /api/v1/invoices/

List invoices.

**Query parameters:**

- `billing_account` (optional): filter by billing account
- `status` (optional): filter by status (draft, finalized)
- `period_start` (optional): filter by period start date
- `period_end` (optional): filter by period end date

**Response:**

- Status 200: List of invoices

**Response body:**

```json
[
  {
    "id": "<id>",
    "invoice_number": null,
    "billing_account": "<id>",
    "period_start": "2026-01-01",
    "period_end": "2026-01-31",
    "status": "draft",
    "total_amount": "1500.50",
    "currency": "NOK",
    "created_at": "2026-01-15T10:30:00Z"
  }
]
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
  "created_at": "2026-01-15T10:30:00Z",
  "finalized_at": null,
  "metadata": {
    "selection_scope": "resource_types",
    "selected_resource_types": ["storage_hotel"],
    "autofill_missing_days": true,
    "force": false,
    "incomplete": false
  },
  "lines": [
    {
      "id": "<id>",
      "resource_type": "storage_hotel",
      "resource_id": 101,
      "description": "StorageHotel #101",
      "total_cost": "1500.50",
      "currency": "NOK",
      "metadata": {
        "billing_dimensions": ["quota_tb"],
        "total_quantity_by_dimension": {
          "quota_tb_days": "31.5"
        },
        "filesystem_identifier": "storage-001"
      }
    }
  ]
}
```

### POST /api/v1/invoices/{id}/finalize

Transition a draft invoice to finalized.

Once finalized, the invoice becomes immutable.

**Request body:**

```json
{}
```

**Response:**

- Status 200: Invoice finalized
- Status 404: Invoice not found
- Status 409: Invoice is already finalized or does not exist
- Status 422: Cannot finalize a non-draft invoice

**Response body:**

```json
{
  "id": "<id>",
  "invoice_number": "INV-2026-01-00001",
  "status": "finalized",
  "finalized_at": "2026-01-15T11:00:00Z",
  "total_amount": "1500.50",
  "currency": "NOK"
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

Invoice generation must fail (return 400 or 409) if:

- selected resources do not belong to the provided billing account
- requested resource types are unknown or unsupported
- the selection is empty or ambiguous
- the same resource is selected more than once
- required pricing data is missing and `force=false`
- required usage snapshots are missing and both `force=false` and `autofill_missing_days=false`
- a matching finalized invoice already exists (immutability rule)
- a matching draft invoice already exists (unless `force=true`)

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

1. Check for a matching finalized invoice with the same `(billing_account, period_start, period_end, selection_scope, selected_resource_types, explicit_resources)`.
   - If found, return 409 Conflict. Finalized invoices are immutable.

2. Check for a matching draft invoice with the same selection parameters.
   - If found and `force=false`: return 409 Conflict.
   - If found and `force=true`: replace the draft invoice atomically.

---

## Force=true Regeneration Rules

When `force=true` is passed:

- If a matching draft invoice exists, it is deleted with all its children (InvoiceLines, InvoiceDailyCosts) in the same transaction, and a new draft is created. The old invoice number (if any) is not reused.
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

## Optional Future Endpoints (v2+)

These endpoints are noted for future development but are not part of v1:

```
GET    /api/v1/invoices/{id}/lines          — retrieve invoice lines in detail
GET    /api/v1/invoices/{id}/daily-costs    — retrieve daily-cost breakdown
POST   /api/v1/invoices/{id}/recalculate    — recalculate a draft (before finalization)
DELETE /api/v1/invoices/{id}                — delete a draft invoice
```

For v1, retrieval of lines is included in the main GET /{id}/ response.
