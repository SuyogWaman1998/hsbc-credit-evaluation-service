# ADR-0008: Timeout, Unknown Outcome and Reconciliation

- **Status:** Accepted
- **Date:** 2026-09-30
- **Decision owners:** CES design team

## Context

An external assessment request can time out after CES has successfully sent the request to the provider. A communication timeout does not prove whether the provider processed the business operation.

For example:

```text
REQUEST_SENT
    ↓
WAITING_FOR_RESPONSE
    ↓
READ_TIMEOUT
    ↓
providerOutcome = UNKNOWN
```

Blindly submitting the same business request again can create duplicate processing when the provider has already accepted or completed the original operation.

CES therefore needs an explicit recovery flow that preserves the original interaction, determines the provider-side outcome where possible, and avoids treating technical uncertainty as a business decision.

## Decision

When a request has been sent to the provider but the expected response is not received within the configured timeout, CES will:

1. Persist the communication evidence for the original interaction.
2. Mark the provider outcome as `UNKNOWN` unless provider evidence establishes another outcome.
3. Move the evaluation/assessment into an appropriate waiting/recovery state rather than immediately resubmitting the business operation.
4. Use provider-supported reconciliation/status mechanisms where available to determine the outcome of the original operation.
5. If reconciliation reports `PROCESSING`, continue asynchronous reconciliation rather than submitting the original business operation again.
6. If reconciliation reports a terminal outcome, persist that outcome and continue the evaluation accordingly.
7. Only retry/resubmit the original business operation when the retry is considered safe according to provider semantics and available idempotency/reconciliation guarantees.

## Reconciliation flow

Conceptually:

```text
Request sent
    ↓
Timeout
    ↓
Communication evidence persisted
    ↓
Provider outcome = UNKNOWN
    ↓
Reconciliation required
    ↓
Provider status lookup
    │
    ├── COMPLETED ───────→ persist result → continue evaluation
    │
    ├── FAILED ──────────→ persist outcome → apply evaluation rules
    │
    ├── NOT_ACCEPTED ────→ persist outcome → apply evaluation rules
    │
    └── PROCESSING ──────→ wait → reconcile again
```

The original business operation is not automatically resubmitted merely because a communication timeout occurred.

## Asynchronous processing

Reconciliation is an asynchronous activity. CES should not hold the original synchronous API request open while waiting for an external provider to finish processing.

Kafka will be used as the asynchronous event transport for reconciliation and resulting evaluation events where appropriate.

Conceptually:

```text
Credit Evaluation
       ↓
Reconciliation required
       ↓
Kafka
       ↓
Reconciliation worker
       ↓
Provider status API
       ↓
Provider result
       ↓
Kafka event
       ↓
CES consumer
       ↓
PostgreSQL
```

Kafka is not the system of record for the evaluation. The durable evaluation/assessment state remains in PostgreSQL.

## `PROCESSING` outcome

If the provider reports that the original operation is still processing:

```text
providerOutcome = PROCESSING
```

CES will keep the evaluation in a waiting/recovery lifecycle state and schedule another reconciliation attempt according to the configured reconciliation policy.

CES will not create a new business assessment merely because another reconciliation attempt occurs.

Each reconciliation communication is represented as an external interaction so that the communication history remains auditable.

## Reconciliation limits

CES must not wait indefinitely for an external operation.

The reconciliation policy will define bounded controls such as:

- maximum reconciliation duration
- maximum reconciliation attempts
- retry/backoff interval
- provider-specific recovery rules

These values are configuration/business-policy concerns and will be finalized during implementation and operational design.

## When the reconciliation limit is exhausted

If CES reaches the configured reconciliation limit without obtaining a reliable terminal provider outcome, the evaluation must not be automatically marked `REJECTED` or `APPROVED`.

Instead, CES will transition the evaluation into a **recovery/manual-review state** with:

```text
providerOutcome = UNKNOWN

decision = null
```

The record must preserve the interaction and reconciliation history so that the operation can be investigated or recovered without losing the original evidence.

The exact status name will be finalized with the evaluation state machine, but the semantic meaning is:

> Required external evidence could not be conclusively obtained within the permitted recovery window.

This is a technical/recovery condition, not a business decision.

## Idempotency

If the provider supports idempotency, CES must use a stable operation/idempotency identifier across retries or recovery attempts so that repeated delivery of the same business operation does not create duplicate provider processing.

For example:

```text
Evaluation: EV-001
Assessment: CA-001
Operation: CREDIT_CHECK
Idempotency key: EV-001-CA-001-CREDIT_CHECK
```

The same logical operation must retain the same identifier when a provider-supported retry is performed.

Idempotency does not by itself prove that an unknown operation completed. It is a mechanism for making repeated requests safe when the provider contract explicitly supports it.

## Retry versus reconciliation

The following rule applies:

```text
Communication timeout after request sent
        ↓
Reconcile first
        ↓
Determine provider outcome where possible
        ↓
Retry only when safe and explicitly supported
```

A retry must not be triggered solely because the client-side timeout elapsed.

## Kafka DLT boundary

A provider operation being `PROCESSING` or having an `UNKNOWN` outcome is **not a Kafka dead-letter condition**.

A Dead Letter Topic is reserved for an asynchronous Kafka message that cannot be successfully consumed after the configured retry policy has been exhausted or is otherwise determined to be non-processable.

For example:

```text
Kafka event
    ↓
CES consumer
    ↓
processing failure
    ↓
configured retries
    ↓
still failing
    ↓
DLT
```

The DLT therefore represents a failure of event processing, not an unknown provider business outcome.

## Alternatives considered

### Immediately retry the original request after timeout

Rejected because the provider may already have processed the original request, creating duplicate business operations.

### Treat timeout as provider failure

Rejected because the timeout only proves that CES did not receive the expected response within the communication window.

### Wait indefinitely for the provider

Rejected because evaluations require bounded operational behavior and indefinite external dependencies can consume resources and leave business processes unresolved.

### Put provider-unknown operations directly into Kafka DLT

Rejected because DLT is intended for failed event consumption, not normal asynchronous reconciliation or external-provider uncertainty.

### Automatically reject after reconciliation timeout

Rejected because technical inability to obtain evidence is not equivalent to a business rejection.

## Consequences

### Positive

- Prevents blind duplicate external operations.
- Preserves evidence of the original communication attempt.
- Separates provider uncertainty from business decisions.
- Allows slow providers to complete asynchronously.
- Provides bounded recovery instead of indefinite waiting.
- Gives operations a durable record for manual recovery when automated reconciliation cannot establish an outcome.
- Keeps Kafka's role focused on asynchronous event transport and DLT-based event recovery.

### Trade-offs

- Requires reconciliation/status support from external providers where available.
- Adds asynchronous processing and scheduling complexity.
- Requires configuration for reconciliation limits and backoff policies.
- Manual recovery procedures are required for unresolved terminal cases.
- Idempotency behavior depends partly on provider capabilities and contracts.

## Related decisions

- `docs/decisions/ADR-0001-documentation-strategy.md`
- `docs/decisions/ADR-0002-service-boundary-and-ownership.md`
- `docs/decisions/ADR-0003-application-snapshot-and-evaluation-time-data.md`
- `docs/decisions/ADR-0004-evaluation-status-vs-business-decision.md`
- `docs/decisions/ADR-0005-assessment-model.md`
- `docs/decisions/ADR-0006-external-service-interaction-and-evidence.md`
- `docs/decisions/ADR-0007-communication-state-vs-provider-operation-outcome.md`
- `docs/architecture/system-design.md`
