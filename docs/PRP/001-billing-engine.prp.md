# Billing Engine Specification

This document defines the **billing algorithm and financial invariants**.

---

# Billing Strategy

Billing is calculated **per resource per day**.

Each invoice stores:

- daily calculations
- applied pricing
- discounts
- final totals

---

# Daily Calculation

For each resource and each day:

1. Load daily snapshot
2. Normalize usage units
3. Determine if discount applies
4. Resolve effective price
5. Compute daily cost

Formula:

```
daily_cost = usage × price_per_year / days_in_year(day)
```

Where:

```
days_in_year(day) = 365 or 366
```

---

# Invoice Total

```
invoice_total = Σ daily_cost
```

---

# Invoice Snapshots

Invoice snapshots must retain enough resource-identifying data to remain understandable after invoice finalization, even if the live resource later changes, is renamed, or is soft-deleted.

For that reason, InvoiceLine.metadata and InvoiceDailyCost.metadata should include a frozen resource snapshot containing the minimal identifying attributes needed for audit and display.

---

# Rounding

Internal calculations:

```
Decimal high precision
```

Customer totals:

```
2 decimals NOK applying only at the invoice total after all calculations
```

Detailed rounding rule:

- `InvoiceDailyCost` rows remain at full `Decimal` precision
- `InvoiceLine.total_cost` remains at full `Decimal` precision
- `Invoice.total_amount` is rounded to 2 decimal places NOK
- Rounding happens once at the invoice level, not per line
- Required rounding method: `ROUND_HALF_UP`

The difference between `Invoice.total_amount` (rounded, 2 decimals) and `InvoiceLine.total_cost` (full precision) is intentional:
- `total_amount` is the customer-visible total
- `total_cost` on InvoiceLine captures the full-precision per-resource cost

---

# Allowed Pricing Dimensions

Define the allowed `pricing_dimension` values:

**StorageHotel:**
- `quota_tb`

**VirtualMachine:**
- `cpu_count`
- `ram_gb`
- `disk_gb`

The billing engine matches `ResourcePrice` rows using `(resource_type, pricing_dimension)`.

---

# Explicit Resource Selection Format

Explicit resource selection must use `(resource_type, resource_id)` pairs rather than a plain list of IDs.

Recommended input shape:

```json
explicit_resources = [
  {"resource_type": "storage_hotel", "resource_id": 101},
  {"resource_type": "virtual_machine", "resource_id": 205}
]
```

This approach is consistent with the existing `resource_type + resource_id` pattern used by `InvoiceLine` and `InvoiceDailyCost`.

---

# Force Mode and Missing Data

When `force=true` and `autofill_missing_days=false`:

Resources with missing days are billed at **zero** for those missing days. The invoice line is included with zero cost.

Missing days must be reported in the invoice generation response.

Draft replacement with `force=true`:

When `force=true` and a matching draft invoice exists, the old draft and all its children (InvoiceLines, InvoiceDailyCosts) are deleted in the same transaction and a new invoice is created. The old invoice number (if any) is not reused. The new draft starts with `invoice_number = null`.

---

# Concurrency and Duplicate Prevention

There must be at most one draft invoice per `(billing_account, period_start, period_end, selection_scope, selected_resource_types, explicit_resources)`.

A matching finalized invoice must block regeneration entirely (finalized invoices are immutable).

A matching draft is replaced atomically when `force=true`.

To prevent race conditions when two invoice generation requests arrive simultaneously for the same `(billing_account, period_start, period_end)`, the system uses a PostgreSQL advisory lock keyed on these three values within the invoice generation transaction. This ensures that the duplicate check and invoice creation are atomic.

---

# Pricing Model

## PriceList

Defines pricing for a BillingAccount.

Each BillingAccount references **exactly one PriceList**.

---

## ResourcePrice

Defines effective-dated pricing.

Fields:

- `price_list` — FK to PriceList, required
- `resource_type` — CharField(max_length=50), required
- `pricing_dimension` — CharField(max_length=50), required
- `price_per_unit_year` — DecimalField(max_digits=14, decimal_places=4), required
- `price_currency` — CharField(max_length=3, default="NOK")
- `discount_price_per_unit_year` — DecimalField(max_digits=14, decimal_places=4), nullable (null = no discount price)
- `discount_threshold_quantity` — DecimalField(max_digits=14, decimal_places=4), nullable (null = discount does not apply)
- `effective_from` — DateField, required
- `effective_to` — DateField, nullable (null = open-ended)
- `created_at` — DateTimeField, auto_now_add

Rules:

- price rows **never updated** — no PATCH or PUT endpoint exists for ResourcePrice
- new pricing → insert new rows with adjusted effective dates
- to correct a price: set `effective_to` on the existing row and create a new row
- `effective_to` is **inclusive** — the price is valid on that day
- `effective_to = null` means open-ended (no expiration)
- No two `ResourcePrice` rows for the same `(price_list, resource_type, pricing_dimension)` may have overlapping effective date ranges — enforced at the service layer
- ResourcePrice is managed via API (see `005-pricing-api.prp.md`)

---

# Discounts

Evaluated **per day**.

Example:

```
threshold = 10 TB
```

Rules:

```
usage >= threshold → discounted price
usage < threshold → normal price

```

If discount_threshold is null, discount does not apply

For multi-dimension resources, the discount threshold on each `ResourcePrice` row is evaluated against the normalized daily usage value of that specific pricing dimension. Each dimension is evaluated independently.

---

# Missing Usage Handling

Default:

```
invoice generation fails
```

Optional flags:

```
force=true
autofill_missing_days=true
```

Autofill rule:

```
carry forward last known value
```

---

# Invoice Lifecycle

States:

```
draft
finalized
```

---

## Invoice Number Assignment

`invoice_number` is assigned at **finalization**, not at draft creation:

- Draft invoices have `invoice_number = null`
- Invoice numbers are assigned during finalization
- Once assigned, an invoice number is immutable and must never be reused
- Gaps in the sequence from abandoned drafts are acceptable under this scheme

---

## Generate draft

Pre-condition: `billing_account.make_invoice` must be `True`. If `make_invoice = False`, invoice generation must fail with a validation error before any resources are evaluated.

Steps:

1. identify billable resources — must satisfy all conditions in the Billable Resource Rule (billing_account not null, `make_invoice = True`, within active window, included by selection)
2. validate usage completeness
3. apply optional autofill
4. compute daily costs
5. create InvoiceDailyCost rows
6. aggregate InvoiceLines

---

## Recalculate

Allowed only if:

```
status == draft
```

---

## Finalize

```
status = finalized
finalized_at = now()
```

Finalized invoices become **immutable**.
