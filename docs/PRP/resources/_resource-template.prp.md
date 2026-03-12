# <ResourceName>

## Purpose

Describe what this resource represents and why it is billable.

---

## Billing role

Explain what daily data is used for invoicing.

---

## Model

### <ResourceName> (inherits ResourceModel)

Fields:

```text
id
billing_account
name
status
created_at
updated_at
deleted_at
```

Resource-specific fields:

```text
<field_1>
<field_2>
```

---

## Daily snapshot model

### <ResourceName>DailyUsage

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
POST   /<resources>/
GET    /<resources>/
GET    /<resources>/{id}/
PATCH  /<resources>/{id}/
POST   /<resources>/{id}/usage
```

---

## Test expectations

- create resource
- ingest daily snapshot
- reject duplicate daily snapshot
- invoice generation works
- pricing changes work
- missing usage handling works
