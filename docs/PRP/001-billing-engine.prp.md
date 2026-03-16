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

**Type coercion rule:** Fields stored as integer types (e.g., `VirtualMachine.cpu_count` as `PositiveIntegerField`) must be coerced to `Decimal` during normalization via `Decimal(str(value))`. This ensures all billing calculations use `Decimal` consistently.

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

# Zero-Billable-Day Exclusion

Resources with zero billable days in the selected period produce no `InvoiceLine` and no `InvoiceDailyCost` rows. They are treated as if they were not selected.

---

# Invoice Total

```
invoice_total = Σ daily_cost
```

---

# Invoice Snapshots

Invoice snapshots must retain enough resource-identifying data to remain understandable after invoice finalization, even if the live resource later changes, is renamed, or is soft-deleted.

`resource_snapshot` is **required** in `InvoiceLine.metadata`. It must contain the minimal identifying attributes needed for audit and display. This snapshot is captured at invoice generation time and remains fixed even if the live resource is later renamed, changed, or soft-deleted.

Each resource PRP defines the exact required `resource_snapshot` schema for that resource type. See `docs/PRP/resources/storage-hotel.prp.md` and `docs/PRP/resources/virtual-machine.prp.md` for the canonical schemas.

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

- `InvoiceDailyCost.daily_cost` uses 10 decimal places (`decimal_places=10`)
- `InvoiceLine.total_cost` uses 10 decimal places (`decimal_places=10`)
- `Invoice.total_amount` is rounded to 2 decimal places NOK
- Rounding happens once at the invoice level, not per line
- Required rounding method: `ROUND_HALF_UP`

The difference between `Invoice.total_amount` (rounded, 2 decimals) and `InvoiceLine.total_cost` (10 decimal places) is intentional:
- `total_amount` is the customer-visible total
- `total_cost` on InvoiceLine captures the 10-decimal-place per-resource cost

---

# Allowed Pricing Dimensions

Define the allowed `pricing_dimension` values:

**StorageHotel:**
- `quota_tb`

**VirtualMachine:**
- `cpu_count`
- `ram_gb` — computed as `ram_mb / 1024` (binary GiB; see `virtual-machine.prp.md` for conversion rules)
- `disk_gb` — passed through 1:1 from `disks_total_gb` with no conversion (unit is as reported by the hypervisor; see `virtual-machine.prp.md`)

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

When `force=true` and a matching draft invoice exists, the old draft and all its children (InvoiceLines, InvoiceDailyCosts) are deleted in the same transaction and a new invoice is created. The replacement draft has `invoice_number = null` (consistent with all draft invoices).

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
- If `force=false`: the **entire invoice generation fails** (fatal -- not a per-resource skip). The entire invoice transaction is rolled back: no invoice is created, no InvoiceLines persist, no InvoiceDailyCost rows persist. Example: if resource A has complete data and resource B has no prior snapshot, and `force=false`, the entire invoice fails -- resource A's valid data does not produce a partial invoice.
- If `force=true`: the resource is billed at zero for all its missing days; reported in `missing_data_summary`; invoice marked `incomplete=true`

Billing at zero is recorded clearly in the invoice metadata so that human review can identify the fallback behavior.

**Invoice flags — `incomplete` vs `provisional`:**

- `incomplete` is a dedicated `BooleanField` on the `Invoice` model (default `false`). It is `true` when `force=true` was used and at least one resource/day could not be resolved from a real snapshot or successful autofill; the system used fallback behavior (e.g., zero-cost billing). An autofilled invoice where all days were successfully filled via carry-forward is **not** incomplete. `Invoice.metadata` may still contain `missing_data_summary` and supporting details when `incomplete=true`.
- `provisional` is stored in `Invoice.metadata` (default `false`). It is `true` when `period_end > today` at the time of generation; the invoice covers future days filled via autofill. An invoice can be both `incomplete=true` and `provisional=true` simultaneously.

---

# Concurrency and Duplicate Prevention

The `Invoice` model includes two new fields that are part of the invoice identity:

- `selection_scope` — CharField, identifies the selection category (e.g., "all_resources", "resource_types", "explicit_resources")
- `selection_fingerprint` — CharField, a deterministic hash of the canonical selection payload used to uniquely identify identical selections

## Fingerprint Specification

`selection_fingerprint` is the SHA-256 lowercase hex digest of the UTF-8-encoded canonical JSON payload.

**Storage:** `CharField(max_length=64)` (SHA-256 hex digest = 64 characters)

**Canonical payload schema (always use all three keys):**

```json
{
  "selection_scope": "<scope>",
  "selected_resource_types": ["sorted", "alphabetically"],
  "explicit_resources": [{"resource_type": "...", "resource_id": 101}]
}
```

**Canonicalization rules:**

- Always include all three keys, even if the value is an empty array
- Stable key ordering: `selection_scope`, `selected_resource_types`, `explicit_resources`
- Sort `selected_resource_types` alphabetically
- Normalize `explicit_resources` to objects with `resource_type` and `resource_id`; sort by `resource_type` then `resource_id`
- Numeric identifiers (`resource_id`) must be serialized as JSON numbers, not strings
- Serialize JSON without insignificant whitespace
- Encode the resulting JSON string as UTF-8 before hashing

**Example for `selection_scope = "all_resources"`:**

```json
{
  "selection_scope": "all_resources",
  "selected_resource_types": [],
  "explicit_resources": []
}
```

## Duplicate Prevention Behavior

There must be at most one draft invoice per `(billing_account, period_start, period_end, selection_scope, selection_fingerprint)`.

A matching finalized invoice must block regeneration entirely (finalized invoices are immutable).

A matching draft is replaced atomically when `force=true`.

**v1 limitation — cross-scope double-billing risk:** v1 does not prevent the same resource from appearing in multiple invoices for the same period under different selection scopes. Operators must not generate invoices with overlapping resource selections for the same billing account and period. Finalization is the operational safeguard — do not finalize overlapping invoices.

## Concurrency Control

To prevent race conditions when two invoice generation requests arrive simultaneously for overlapping invoice identities, the system uses a PostgreSQL advisory lock keyed on `(billing_account, period_start, period_end, selection_scope)` within the invoice generation transaction.

This lock scope is intentionally broader than the full uniqueness constraint `(billing_account, period_start, period_end, selection_scope, selection_fingerprint)` but narrower than locking on account+period only. Including `selection_scope` reduces unnecessary serialization when the same billing account commonly generates separate invoices for different resource categories (e.g., StorageHotel vs VirtualMachine). Requests with the same `selection_scope` but different underlying selections may still serialize. This is a v1 balance between correctness, simplicity, and practical concurrency. `selection_fingerprint` is kept out of the lock key unless a stronger concurrency need appears later.

Inside the locked transaction:

1. Check for a finalized invoice matching `(billing_account, period_start, period_end, selection_scope, selection_fingerprint)` → if found, fail
2. Check for a draft invoice matching the same tuple → if found and `force=false`, fail; if found and `force=true`, replace atomically

---

# Pricing Model

## PriceList

Defines pricing for a BillingAccount.

Each BillingAccount references **exactly one PriceList**.

The billing engine resolves the price list from the account's **current** FK at invoice-generation time. Historical price-list assignment is not tracked in v1. If `BillingAccount.price_list` is changed after historical usage has been captured, future invoice generation for uninvoiced periods will use the current `price_list`. Previously generated draft invoices for affected periods should be regenerated before finalization. Already finalized invoices remain immutable. This should be revisited if historical price-list assignment tracking becomes necessary.

---

## ResourcePrice

Defines effective-dated pricing.

Fields:

- `price_list` — FK to PriceList, required
- `resource_type` — CharField(max_length=50), required
- `pricing_dimension` — CharField(max_length=50), required
- `price_per_unit_year` — DecimalField(max_digits=14, decimal_places=4), required
- `price_currency` — CharField(max_length=3, default="NOK"). **v1 constraint:** All pricing in v1 uses NOK. The billing engine must validate that all resolved `ResourcePrice` rows for an invoice use the same currency as `Invoice.currency`. A currency mismatch is a fatal billing error.
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

If discount_threshold is null, discount does not apply.

**Cross-field invariant:** `discount_price_per_unit_year` and `discount_threshold_quantity` must both be set or both be null. This is enforced at ingestion time (ResourcePrice creation). The billing engine assumes this invariant holds for all resolved price rows.

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

---

## `total_quantity_by_dimension` Computation

`InvoiceLine.metadata.total_quantity_by_dimension` aggregates the daily billing snapshots for a resource within an invoice.

**Formal definition:** `{dimension}_days` is the sum of `InvoiceDailyCost.metadata.normalized_usage[dimension]` across all billed days for the same `(invoice, resource_type, resource_id)` tuple. Autofilled billed days are included in this sum. Non-billable days are excluded because no `InvoiceDailyCost` row exists for them.

This makes the line-level quantity summary deterministic, explainable, and reproducible from persisted `InvoiceDailyCost` rows.

---

## InvoiceLine / InvoiceDailyCost Tuple Integrity Invariant

Invoice generation must guarantee tuple-level consistency for `(invoice, resource_type, resource_id)`.

For every such tuple in a committed invoice:

- if one or more `InvoiceDailyCost` rows exist, exactly one corresponding `InvoiceLine` must exist
- if an `InvoiceLine` exists, it must aggregate all `InvoiceDailyCost` rows for that same tuple
- invoice generation must run inside a transaction so partial writes cannot leave orphaned `InvoiceDailyCost` rows or unmatched `InvoiceLine` rows

This invariant is enforced by the transaction boundary, not by a direct FK between `InvoiceDailyCost` and `InvoiceLine`.
