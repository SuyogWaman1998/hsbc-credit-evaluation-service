# ADR-0004: Evaluation Status vs Business Decision

- **Status:** Accepted
- **Date:** 2026-09-23
- **Decision owners:** CES design team

## Context

A credit evaluation has two different dimensions that must not be conflated:

1. The **processing/lifecycle state** of the evaluation.
2. The **business decision** produced by the evaluation.

An evaluation can be received, processing, waiting for an external assessment, or blocked by a technical issue without having a business decision. Likewise, `APPROVED` and `REJECTED` describe business outcomes rather than technical processing states.

Using one field for both concepts would make it difficult to distinguish an incomplete evaluation from a completed evaluation whose business outcome is rejection.

## Decision

CES will maintain separate `status` and `decision` concepts on the evaluation.

### Evaluation status

`status` describes where the evaluation currently is in its processing lifecycle.

The status must answer:

> "What is happening with this evaluation right now?"

The exact final state machine will be finalized as the orchestration design is completed, but the model must distinguish lifecycle/progress states from business outcomes.

### Business decision

`decision` describes the business outcome reached after the required evaluation processing and rules have been completed.

Examples include:

```text
APPROVED
REJECTED
MANUAL_REVIEW
```

The final decision vocabulary is subject to the business requirements.

The decision must answer:

> "What business outcome did the evaluation produce?"

## Example

An evaluation may move through a lifecycle such as:

```text
RECEIVED
   ↓
PROCESSING
   ↓
WAITING_FOR_CREDIT_ASSESSMENT
   ↓
PROCESSING
   ↓
COMPLETED
```

At completion, the business decision may be:

```text
decision = APPROVED
```

Another evaluation may also reach:

```text
status = COMPLETED
decision = REJECTED
```

The important distinction is that `REJECTED` is not itself a processing status.

## Technical failure is not a business rejection

A technical failure must not automatically become a rejection decision.

For example:

```text
status   = BLOCKED / FAILED / equivalent technical lifecycle state
decision = null
```

The exact technical failure-state vocabulary will be finalized with the failure-handling state machine.

This preserves the distinction between:

```text
Business outcome:
"The applicant was rejected by the evaluation rules."

Technical outcome:
"The evaluation could not complete because required processing failed."
```

## Alternatives considered

### Use one field for both status and decision

Rejected because a business rejection and a technical/in-progress state represent fundamentally different concepts and have different consumers and operational meanings.

### Use `APPROVED` / `REJECTED` as evaluation statuses

Rejected because those values describe the business decision, not the lifecycle state of processing.

### Store decision only and infer processing state

Rejected because absence of a decision cannot reliably explain whether an evaluation is still processing, waiting on an external service, failed, or has not started the relevant stage.

## Consequences

### Positive

- Business outcome and processing state remain unambiguous.
- API consumers can distinguish an incomplete evaluation from a completed rejection.
- Technical failures are not incorrectly represented as business outcomes.
- Database constraints and state transitions can be designed around clear responsibilities.
- Operational monitoring can focus on lifecycle status while business reporting can focus on decision.

### Trade-offs

- CES maintains two pieces of state instead of one.
- The relationship between status and decision must be explicitly defined.
- The state machine needs validation so that invalid combinations cannot be persisted as valid completed evaluations.

## Data model implication

Conceptually:

```text
CreditEvaluation
├── status    ← evaluation lifecycle
└── decision  ← business outcome
```

`decisionReason` belongs to the business decision context and should explain the reason for the resulting decision where applicable.

## Related decisions

- `docs/decisions/ADR-0001-documentation-strategy.md`
- `docs/decisions/ADR-0002-service-boundary-and-ownership.md`
- `docs/decisions/ADR-0003-application-snapshot-and-evaluation-time-data.md`
- `docs/architecture/system-design.md`
