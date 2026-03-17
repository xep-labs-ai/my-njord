# Invoice API — System Overview

## Owner
Engineering (Django API)

---

# One-liner

Build a **Django REST API** that ingests **daily resource snapshots** and generates **reproducible invoices** for company IT resources using **effective-dated pricing** and **daily cost calculation**.

Initial supported resources:

- StorageHotel (filesystem quota)
- VirtualMachine (compute resource)

---

# System Philosophy

The system invoices **IT resources owned by billing entities**.

A resource is something that:

- belongs to a **BillingAccount**
- has a **lifecycle**
- produces **daily measurable data**
- can participate in **invoicing**

All billable resources inherit from:

```
ResourceModel
```

Future examples:

- CPUAllocation
- MemoryAllocation
- GPUResource
- BackupResource
- NetworkAllocation

---

# Primary Goals

## G1 — Resource-centric billing

Billing is based on **resources**, not services.

Each resource belongs to a **BillingAccount** and generates **daily usage data**.

---

## G2 — Daily snapshot ingestion

Resources produce **daily immutable snapshots**.

Rules:

- max **1 snapshot per resource per day**
- snapshots are **append-only**
- ingestion happens via **API push**

---

## G3 — Effective-dated pricing

Prices may change during the year.

Example:

```
01-01-2026 → 30-05-2026
500 NOK / TB / year

01-06-2026 → ∞
600 NOK / TB / year
```

Invoice calculation automatically selects the **correct price per day**.

---

## G4 — Arbitrary billing ranges

Invoices may cover **any date range**.

Examples:

```
01-01-2026 → 31-01-2026
04-02-2026 → 08-11-2026
15-03-2026 → 21-03-2026
```

---

## G5 — Reproducible invoices

Once finalized:

- invoice totals **never change**
- pricing snapshots are stored
- daily calculations are stored

This guarantees **auditability**.

---

# Non-Goals (v1)

- authentication / authorization
- PDF invoice generation
- external accounting integrations
- advanced compute billing formulas

---

# Definitions

## BillingAccount

Represents the **billing entity**.

Examples:

- organization
- department
- project

Each BillingAccount uses **exactly one PriceList**.

---

## Resource

A **billable IT entity**.

Examples:

- StorageHotel
- VirtualMachine

Lifecycle:

```
UNASSIGNED
ACTIVE
RETIRED
```

Billing rule (per day):

```
billing_account IS NOT NULL
AND active_from <= day
AND (active_to IS NULL OR day <= active_to)
```

**Important:** `status` is for operational and display purposes only. It does not determine billing eligibility.

Billing eligibility is determined exclusively by `active_from` and `active_to` date ranges:

- `status` can be `UNASSIGNED`, `ACTIVE`, or `RETIRED` — these represent the resource's current lifecycle state
- Setting `status = RETIRED` does not retroactively affect billing
- Billing stops when `active_to` is reached, not when status changes
- A resource with `status = RETIRED` and a future `active_to` date remains billable until that date

Example: a resource with `status = RETIRED` and `active_to = 2026-12-31` remains billable through 2026-12-31 even though its status changed.

Note: `billing_account.make_invoice = True` is a pre-condition checked before resource evaluation, not a per-day condition. See `001-billing-engine.prp.md` for the authoritative billing rule.

---

# Time Rules

Timezone:

```
Europe/Oslo
```

All calculations are **date-based**.

---

# Data Model Overview

Main Django apps:

```
apps/
 ├── billing/
 └── ingest/
```

Detailed resource models are defined in:

```
002-resource-models.prp.md
```

Billing algorithm is defined in:

```
001-billing-engine.prp.md
```

---

# Project Structure

```
apps/<app>/
├── api/
├── migrations/
├── models/
├── selectors/
├── tests/
└── services/
```

---

# Configuration Layout

```
config/
├── settings/
│   ├── base.py
│   ├── dev.py
│   ├── test.py
│   └── prod.py
├── urls.py
├── asgi.py
└── wsgi.py
```

`config/settings/test.py` is used for automated tests (`DJANGO_SETTINGS_MODULE = "config.settings.test"` in `pyproject.toml`). `config/settings/dev.py` is for local development only.

---

# Performance Target

Expected scale:

```
~10,000 resources
```

Primary heavy operation:

```
invoice generation
```

Processing should be **batched**.
