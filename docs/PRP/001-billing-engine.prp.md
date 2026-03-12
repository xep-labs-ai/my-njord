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

# Rounding

Internal calculations:

```
Decimal high precision
```

Customer totals:

```
2 decimals NOK
```

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
price_nok_per_unit_year
discount_price_nok_per_unit_year
discount_threshold
effective_from
effective_to
```

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
