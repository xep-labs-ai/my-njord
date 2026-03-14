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

---

# Duplicate Invoice Prevention

There must be at most one draft invoice per `(billing_account, period_start, period_end, selection_scope, selected_resource_types, explicit_resources)`.

A matching finalized invoice must block regeneration entirely (finalized invoices are immutable).

A matching draft is replaced atomically when `force=true`.

---

# Pricing Model

## PriceList

Defines pricing for a BillingAccount.

Each BillingAccount references **exactly one PriceList**.

---

## ResourcePrice

Defines effective-dated pricing.

Fields:

```
price_list
resource_type
pricing_dimension

price_per_unit_year
price_currency

discount_price_per_unit_year
discount_threshold_quantity

effective_from
effective_to```

Rules:

- price rows **never updated**
- new pricing → insert new rows

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

## Generate draft

Steps:

1. identify active resources
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
