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

Daily cost calculation is **resource-type specific**. There is no universal formula applied to all resources.

For resource types with yearly-prorated pricing, an illustrative formula is:

```
daily_cost = usage × price_per_year / days_in_year(day)
```

Where:

```
days_in_year(day) = 365 or 366
```

For multi-dimension resources (e.g., VirtualMachine), the per-dimension daily costs are summed to produce the final `InvoiceDailyCost.daily_cost`:

```
InvoiceDailyCost.daily_cost = Σ dimension_daily_cost
```

Each resource type is responsible for defining its own daily cost formula, subject to the constraint that the result is deterministic and reproducible from persisted invoice snapshot data.

---

# Invoice Total

```
invoice_total = Σ daily_cost
```

---

# Invoice Snapshots

Invoice snapshots must retain enough resource-identifying data to remain understandable after invoice finalization, even if the live resource later changes, is renamed, or is soft-deleted.

`resource_snapshot` is **required** in `InvoiceLine.metadata`. It must contain the minimal identifying attributes needed for audit and display. This snapshot is captured at invoice generation time and remains fixed even if the live resource is later renamed, changed, or soft-deleted.

`InvoiceDailyCost.metadata` may optionally include `resource_snapshot`; it is not required at the daily-cost level — the InvoiceLine-level snapshot is sufficient for auditability.

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

# Resource Type Registry

Valid `resource_type` values:

| Value | Django Model |
|---|---|
| `"storage_hotel"` | `StorageHotel` |
| `"virtual_machine"` | `VirtualMachine` |

Naming convention: snake_case of the Django model name.

Code location: a choices class or constant module in `apps/billing/` (e.g., `apps/billing/resource_types.py`).

Validation: invoice generation and `ResourcePrice` creation must reject unknown `resource_type` values with **400 Bad Request**.

New resource types must be registered here before they can be billed.

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

**Missing pricing is always fatal:**

Missing pricing data causes invoice generation to fail even when `force=true`. The `force` flag only affects missing usage data and duplicate draft handling. A resource billed at zero due to a pricing configuration gap is more dangerous than a failed invoice.

**Future-dated invoice periods:**

By default, `period_end > today` is rejected with 400 Bad Request. Invoice periods must lie entirely in the past or current day unless `autofill_missing_days=true` is explicitly set.

When `period_end > today` and `autofill_missing_days=true`:
- Invoice generation is allowed using the standard autofill carry-forward rule
- Future days have no real snapshots (snapshot ingestion rejects future-dated data), so all future days are filled by carrying forward the last known value
- The invoice is marked `provisional = true` in metadata
- If no prior snapshot exists for a required resource, normal missing-data policy applies
- `force=true` alone does not permit future-dated periods; `autofill_missing_days=true` is required

**Autofill with no prior snapshot:**

When `autofill_missing_days=true` and no prior snapshot exists for a resource:
- If `force=false`: entire invoice generation fails (fatal)
- If `force=true`: the resource is billed at zero for all its missing days; reported in `missing_data_summary`; invoice marked `incomplete=true`

Billing at zero is recorded clearly in the invoice metadata so that human review can identify the fallback behavior.

**Invoice metadata flags — `incomplete` vs `provisional`:**

These are independent flags in `Invoice.metadata`:

- `incomplete = true` — `force=true` was used and at least one resource/day could not be resolved from a real snapshot or successful autofill; the system used fallback behavior (e.g., zero-cost billing). An autofilled invoice where all days were successfully filled via carry-forward is **not** incomplete.
- `provisional = true` — `period_end > today` at the time of generation; the invoice covers future days filled via autofill. An invoice can be both `incomplete=true` and `provisional=true` simultaneously.

---

# Concurrency and Duplicate Prevention

The `Invoice` model includes two new fields that are part of the invoice identity:

- `selection_scope` — CharField, identifies the selection category (e.g., "all_resources", "resource_types", "explicit_resources")
- `selection_fingerprint` — CharField, a deterministic hash of the canonical selection payload used to uniquely identify identical selections

## Canonicalization Rules for Fingerprint Computation

Before hashing:

- `resource_types` must be sorted alphabetically
- `explicit_resources` must be normalized to `(resource_type, resource_id)` tuples and sorted deterministically (by resource_type first, then by resource_id)
- `selection_scope` is included in the hash input

## Duplicate Prevention Behavior

There must be at most one draft invoice per `(billing_account, period_start, period_end, selection_scope, selection_fingerprint)`.

A matching finalized invoice must block regeneration entirely (finalized invoices are immutable).

A matching draft is replaced atomically when `force=true`.

## Concurrency Control

To prevent race conditions when two invoice generation requests arrive simultaneously for the same `(billing_account, period_start, period_end)`, the system uses a PostgreSQL advisory lock keyed on these three values within the invoice generation transaction.

Inside the locked transaction:

1. Check for a finalized invoice matching `(billing_account, period_start, period_end, selection_scope, selection_fingerprint)` → if found, fail
2. Check for a draft invoice matching the same tuple → if found and `force=false`, fail; if found and `force=true`, replace atomically

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
- No two `ResourcePrice` rows for the same `(price_list, resource_type, pricing_dimension)` may have overlapping effective date ranges — validated at the service layer and enforced by a PostgreSQL `daterange` exclusion constraint on the database
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

**Generation algorithm:**

- Format: `INV-YYYY-mm-NNNNN`
- `YYYY-mm` is derived from the **finalization date** (not `period_start`) — the number reflects when the invoice was issued
- `NNNNN` is a **global auto-incrementing sequence** (not per-month) — the counter does not reset each month
- Concurrency: use a dedicated PostgreSQL sequence or `SELECT MAX(invoice_number) FOR UPDATE` within the finalization transaction to prevent duplicate assignment

---

## Generate draft

Pre-flight validation (happens before any resource selection or per-day evaluation):

- `billing_account.make_invoice` must be `True`. If `make_invoice = False`, invoice generation must fail immediately with a validation error.

Resource identification and selection:

Steps:

1. identify billable resources — must satisfy all conditions in the Billable Resource Rule (billing_account not null, within active window, included by selection)
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
