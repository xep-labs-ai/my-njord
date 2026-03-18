# <ResourceName>

## Purpose

Describe what this resource represents and why it is billable.

---

## Billing role

Explain what daily data is used for invoicing.

---

## Model

### <ResourceName> (inherits ResourceModel)

Fields (inherited from ResourceModel):

```text
id
billing_account
name
description_resource
status
active_from
active_to
deleted_at
created_at
updated_at
```

Resource-specific fields:

```text
<field_1>
<field_2>
```

---

## Daily snapshot model

### <ResourceName>DailyUsage

Note: `<ResourceName>DailyUsage` is a suggested naming convention. Resource-specific suffixes (e.g., `DailyQuota`, `DailyCapacity`) are acceptable when the domain term is clearer than the generic `DailyUsage`.

Fields:

```text
id
<resource_fk>
date
<usage_field_1>
<usage_field_2>
created_at
```

Constraint:

```text
(<resource_fk>_id, date) UNIQUE
```

---

## Ingestion event model

### <ResourceName>UsageIngestionEvent

Fields:

```text
id
<resource_fk>
date
raw_payload
<normalized_field_1>
<normalized_field_2>
request_id
created_at
```

---

## Billing unit

Define the normalized billing unit for this resource.

---

## Unit Conversion Rules

Define any raw-to-billing-unit conversions for this resource.

Pattern: specify the conversion formula from each raw ingestion field to the corresponding billing dimension. Example: `raw_field → billing_dimension: conversion formula`.

---

## Manager Availability

`<ResourceName>` inherits the `billing_objects` manager from ResourceModel. This manager includes soft-deleted resources, which is necessary for the billing engine to bill historical periods.

---

## Soft-Delete Invariants

- If `deleted_at` is set, `status` must be `RETIRED`
- If `deleted_at` is set, `active_to` must be set
- `active_to` must be on or before the calendar date of `deleted_at`
- Default querysets exclude soft-deleted resources; use `billing_objects` manager for billing operations
- Billability for historical days is resolved from `active_from`/`active_to`, not from `deleted_at` alone

---

## Autofill Rule

Define the autofill carry-forward behavior for this resource.

Pattern: when autofill is needed for a missing day, all billing-relevant fields are carried forward together from the last known complete snapshot row. Individual fields are never autofilled independently.

---

## Canonical `resource_snapshot` Schema

The `resource_snapshot` in `InvoiceLine.metadata` for `<ResourceName>` must contain:

```json
{
  "id": "<int>",
  "name": "<str>",
  "description_resource": "<str>",
  "<resource_specific_field_1>": "<type>",
  "<resource_specific_field_2>": "<type>"
}
```

This snapshot is frozen at invoice generation time and must be present for all `<ResourceName>` InvoiceLines.

---

## Validation rules

- one snapshot per day
- snapshots are immutable
- ingestion events are append-only

---

## Invoice line expectations

Explain what should appear in `InvoiceLine.metadata` and what aggregation fields matter.

---

## Daily breakdown expectations

Explain what should be stored in `InvoiceDailyCost` for this resource.

---

## API endpoints

```text
POST   /api/v1/<resources>/
GET    /api/v1/<resources>/
GET    /api/v1/<resources>/{id}/
PATCH  /api/v1/<resources>/{id}/
POST   /api/v1/<resources>/{id}/<ingestion-endpoint>
POST   /api/v1/<resources>/{id}/soft-delete
```

Note: The ingestion endpoint name (`/usage` above is a placeholder) is resource-specific and must be defined per resource PRP. Each resource PRP must define its own ingestion endpoint name that reflects the type of billing data being ingested. Prefer singular domain-specific nouns (e.g., `/quota`, `/usage`, `/capacity`).

---

## Test expectations

- create resource
- ingest daily snapshot
- reject duplicate daily snapshot
- invoice generation works
- pricing changes work
- missing usage handling works
