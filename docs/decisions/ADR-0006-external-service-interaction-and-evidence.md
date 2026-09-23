# ADR-0006: External Service Interaction and Evidence

- **Status:** Accepted
- **Date:** 2026-09-23
- **Decision owners:** CES design team

## Context

CES depends on external services such as credit bureaus and other assessment providers. The business assessment produced by such a provider is not sufficient by itself for a production-grade evaluation record.

For auditability and failure handling, CES also needs to know what happened during communication with the external service: when the interaction started, whether the request was sent, whether a response was received, when a failure occurred, and at which stage the interaction stopped.

A single assessment may involve more than one physical communication attempt. A timeout does not necessarily prove that the provider failed to process the request. Therefore, communication evidence must be represented separately from the business assessment.

## Decision

CES will maintain an `ExternalServiceInteraction` concept associated directly with the `CreditEvaluation`, with an optional reference to the related assessment when applicable.

Conceptually:

```text
CreditEvaluation
    │
    ├── Assessment
    │
    ├── ExternalServiceInteraction 1
    ├── ExternalServiceInteraction 2
    └── ExternalServiceInteraction 3
```

The interaction record represents one physical communication attempt or externally observable interaction. It is not itself the business assessment.

## Ownership

The interaction belongs to the evaluation because the communication occurred as part of that evaluation.

An interaction may reference the assessment it supports:

```text
CreditEvaluation
    │
    ├── CreditAssessment CA-001
    │
    └── ExternalServiceInteraction
            └── assessmentId = CA-001
```

The assessment reference is optional because not every interaction necessarily maps directly to one assessment result. For example, an interaction may be part of reconciliation or status lookup.

## Evidence to persist

The interaction record should preserve the minimum durable evidence required to reconstruct what CES knows about the communication.

The evidence should include, where applicable:

- evaluation reference
- assessment reference when applicable
- external provider/service
- operation type
- interaction/attempt identifier
- start timestamp
- request-sent timestamp
- response-received timestamp
- failure timestamp
- stage at which the interaction stopped or failed
- communication state
- provider operation outcome when known
- provider/reference identifier when available
- relevant technical error classification

The exact column names and constraints will be finalized in the database design.

## What should not be treated as business assessment data

An HTTP request attempt, timeout, connection failure, or retry is not automatically a new assessment.

For example:

```text
CreditAssessment CA-001
        │
        ├── Interaction #1 → request sent → timeout
        │
        └── Interaction #2 → request sent → response received
```

The interactions describe the communication history. The assessment describes the resulting business information used by CES.

## Persistence rule

CES should persist the durable interaction record when the relevant response/failure outcome is known or when sufficient failure evidence has been captured to represent the completed interaction attempt.

The design does not require creating a business assessment record merely because an outbound request was initiated.

The exact transaction boundaries and crash-recovery behavior will be finalized in the implementation design.

## Audit principle

For audit purposes, CES should be able to answer at least:

1. Which evaluation was being processed?
2. Which external service was contacted?
3. Which operation was attempted?
4. When did the interaction start?
5. Was the request sent?
6. When was a response received, if any?
7. If it failed, at what stage did it fail?
8. What communication evidence does CES have?
9. What provider-side outcome is known?

The evidence should distinguish facts observed by CES from conclusions about what happened inside the provider.

## Relationship to timeout and unknown outcome

A timeout is a communication event, not automatically a provider business failure.

Example:

```text
requestSentAt      = 10:01:00
responseReceivedAt = null
failedAt           = 10:01:30
failureStage       = WAITING_FOR_RESPONSE
failureType        = READ_TIMEOUT
providerOutcome    = UNKNOWN
```

This records the evidence available to CES without falsely claiming that the provider did or did not process the request.

## Alternatives considered

### Store only the final assessment

Rejected because the final assessment does not provide sufficient evidence for communication failures, retries, timeouts, or audit investigation.

### Store every communication detail directly on the assessment

Rejected because communication history and business assessment have different lifecycles and semantics. Multiple attempts can occur without producing multiple business assessments.

### Store interaction data only in application logs

Rejected as the sole mechanism because logs are operational evidence rather than the durable business/audit record for an evaluation. Logs may also have different retention, querying, and correlation characteristics.

## Consequences

### Positive

- Preserves a durable communication history for evaluation processing.
- Separates business assessment from technical communication attempts.
- Supports investigation of timeout and retry scenarios.
- Provides evidence for audit without assuming provider-side outcomes that CES cannot prove.
- Allows reconciliation/status interactions to be recorded without creating artificial assessments.

### Trade-offs

- Adds a persistence model and additional records.
- Requires careful handling of potentially sensitive request/response information.
- Interaction retention and payload-retention rules must be defined with security, compliance, and audit requirements.
- Additional correlation information is required for operational troubleshooting.

## Related decisions

- `docs/decisions/ADR-0001-documentation-strategy.md`
- `docs/decisions/ADR-0002-service-boundary-and-ownership.md`
- `docs/decisions/ADR-0003-application-snapshot-and-evaluation-time-data.md`
- `docs/decisions/ADR-0004-evaluation-status-vs-business-decision.md`
- `docs/decisions/ADR-0005-assessment-model.md`
- `docs/architecture/system-design.md`
