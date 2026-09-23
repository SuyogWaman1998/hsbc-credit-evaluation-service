# ADR-0005: Assessment Model

- **Status:** Accepted
- **Date:** 2026-09-23
- **Decision owners:** CES design team

## Context

A credit evaluation is composed of multiple distinct assessment concerns. Customer validation, credit assessment, and fraud assessment are different processing stages with different inputs, external dependencies, business meaning, and evidence.

Treating all assessment information as columns directly on `CreditEvaluation` would make the evaluation record increasingly wide and would mix the evaluation's lifecycle with the details produced by individual assessment stages.

The design also needs to preserve the fact that assessment processing may involve more than one attempt or result over the lifetime of an evaluation. A logical assessment concern is not necessarily identical to one physical execution or one external communication attempt.

## Decision

CES will model the major assessment concerns as separate assessment records associated with a `CreditEvaluation`.

The initial assessment concepts are:

```text
CreditEvaluation
    │
    ├── CustomerValidation
    ├── CreditAssessment
    └── FraudAssessment
```

Each assessment has its own identity and stores the business-relevant assessment result/details for that assessment type.

## What an assessment represents

An assessment represents the **business/analytical result for one assessment concern within an evaluation**.

For example:

```text
CreditEvaluation EV-001
        │
        └── CreditAssessment CA-001
                ├── assessment status/result
                ├── normalized credit information
                └── assessment details
```

The assessment is not merely a log entry for an HTTP call. Communication evidence is represented separately by `ExternalServiceInteraction`.

Therefore:

```text
Assessment
    = what the evaluation learned / concluded for a concern

ExternalServiceInteraction
    = how CES communicated with an external provider
```

## Assessment identity and repeated processing

A logical assessment concern belongs to an evaluation, but a real production workflow may need to execute or refresh that assessment more than once.

For example, a credit assessment might be re-run after a provider timeout is reconciled, after required input is corrected, or as part of an explicitly supported re-evaluation flow.

Therefore, the model must not assume that every physical execution or external call is the same thing as the assessment itself.

Where repeated assessment results are required, the exact versioning/history model will be finalized as part of the orchestration and persistence design. The important boundary is that the assessment remains a business result, while communication attempts remain interaction records.

## Why separate assessment records

Separate records provide clear ownership of assessment-specific data and prevent `CreditEvaluation` from becoming a collection of unrelated provider/business columns.

Conceptually:

```text
credit_evaluation
        │
        ├── customer_validation
        │
        ├── credit_assessment
        │
        └── fraud_assessment
```

The detailed schema and fields are intentionally not finalized by this ADR.

## Relationship to the evaluation

An assessment should not have an independent business lifecycle outside the evaluation to which it belongs.

The relationship is conceptually:

```text
CreditEvaluation 1 ─────── * Assessment
```

The concrete cardinality and uniqueness constraints for each assessment type will be finalized with the database model. The design must preserve the invariant that an assessment belongs to an evaluation.

## Assessment details versus evaluation-level data

Assessment-specific data belongs with its assessment rather than being flattened into `CreditEvaluation` when it has no evaluation-level meaning.

For example:

```text
CreditEvaluation
├── evaluationId
├── applicationId
├── customerId
├── loanProduct
├── requestedAmount
├── status
├── decision
└── decisionReason

CreditAssessment
├── assessmentId
├── evaluationId
└── assessment-specific details

FraudAssessment
├── assessmentId
├── evaluationId
└── assessment-specific details
```

The exact fields will be derived from the business rules and provider contracts.

## Assessment result and external response

CES should normalize the information needed for its own evaluation logic rather than coupling the core assessment model directly to a provider's response schema.

Where complete provider responses must be retained for audit or investigation, they belong to the external interaction/evidence design rather than being indiscriminately flattened into the assessment table.

## Alternatives considered

### Store all assessment details directly on CreditEvaluation

Rejected because it mixes evaluation-level data with assessment-specific data, creates a wide and tightly coupled model, and makes future assessment types harder to introduce cleanly.

### Use one generic Assessment table containing every possible field

Not selected as the current design because the different assessment concerns have different semantics and data structures. A generic model could become a sparse collection of unrelated fields or provider-specific data.

The final persistence representation may still use a common abstraction where appropriate, but the domain must preserve the distinction between assessment types.

### Treat every external call as an assessment

Rejected because an external call is a communication event, not necessarily a business result. Retries, timeouts, reconciliation calls, and status lookups must not automatically create separate business assessments.

## Consequences

### Positive

- Clear separation between evaluation-level and assessment-specific data.
- Assessment-specific records are easier to reason about and audit.
- New assessment concerns can be introduced without continually expanding `CreditEvaluation`.
- External communication history remains separate from business assessment results.
- Assessment data can evolve independently where appropriate.

### Trade-offs

- Queries that need complete evaluation information may require joins across assessment tables.
- The database model contains more tables and relationships.
- The orchestration layer must define how assessment lifecycle and history are managed.
- The exact rules for multiple assessment results still need to be finalized.

## Related decisions

- `docs/decisions/ADR-0001-documentation-strategy.md`
- `docs/decisions/ADR-0002-service-boundary-and-ownership.md`
- `docs/decisions/ADR-0003-application-snapshot-and-evaluation-time-data.md`
- `docs/decisions/ADR-0004-evaluation-status-vs-business-decision.md`
- `docs/architecture/system-design.md`
