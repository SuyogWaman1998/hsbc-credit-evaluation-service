# ADR-0003: Application Snapshot and Evaluation-Time Data

- **Status:** Accepted
- **Date:** 2026-09-23
- **Decision owners:** CES design team

## Context

The Application Service is the system of record for the live loan application. During credit evaluation, however, CES needs a stable representation of the application information that was actually supplied to and used by the evaluation.

The live application may change after an evaluation starts. If CES depended only on the current application state, it could become difficult to establish which application data was used to produce a particular evaluation result. At the same time, making CES the owner of the complete application would violate the service boundary established by ADR-0002.

The evaluation therefore needs its own evaluation-time representation of the relevant application data.

## Decision

CES will persist an **application snapshot as part of the CreditEvaluation record**.

The snapshot contains only the application information required by CES for that evaluation and for the associated audit/reproducibility requirements. It is not a second live application record.

Conceptually:

```text
Application Service
        │
        │ evaluation request
        │ application data required by CES
        ▼
Credit Evaluation
        │
        └── Evaluation-time application snapshot
```

The snapshot is created when the evaluation is created from the data supplied in the initial evaluation request.

Once attached to an evaluation, the snapshot is treated as immutable for that evaluation. Subsequent changes to the live application do not mutate the historical evaluation snapshot.

## Ownership

The Application Service remains the owner of the live application.

CES owns the snapshot because it is part of the evaluation record and exists to establish the input context for that evaluation.

Conceptually:

```text
Live application
    └── owned by Application Service

Evaluation snapshot
    └── owned by Credit Evaluation
```

## Data scope

CES should persist only the application fields that are relevant to evaluation processing, decision-making, traceability, or required audit evidence.

It should not copy the complete application indiscriminately merely because the source system contains additional fields.

The exact snapshot fields will be finalized together with the evaluation data model and business/risk rules.

## Alternatives considered

### Store no application snapshot and retrieve the live application when needed

Rejected as the primary design because the live application can change after evaluation starts. Retrieving current data later would not necessarily reproduce the input context used by the original evaluation.

### Store the entire application as a duplicate of the Application Service

Rejected because it duplicates data outside CES's responsibility and creates unnecessary ownership ambiguity.

### Store the snapshot as an independent top-level entity

Not selected. The snapshot has no meaningful business existence outside the evaluation that captured it. Keeping it as part of `credit_evaluation` makes the ownership and lifecycle explicit and prevents an orphan snapshot from existing without an evaluation.

## Consequences

### Positive

- Preserves the evaluation-time input context.
- Keeps the Application Service as the source of truth for the live application.
- Prevents later application changes from altering the historical evaluation context.
- Avoids creating an independent snapshot lifecycle.
- Keeps the amount of duplicated data controlled by explicitly selecting evaluation-relevant fields.

### Trade-offs

- Some application data is intentionally duplicated between services.
- Changes to the evaluation input contract may require schema/model changes in CES.
- The snapshot must be handled carefully because it may contain sensitive application/customer information.

## Relationship to assessment and decision

The snapshot represents the input context for the evaluation. Assessments and the final decision represent processing/results derived from that context.

```text
Evaluation
   │
   ├── Application snapshot  ← input context
   │
   ├── Assessments           ← evaluation evidence/results
   │
   └── Decision              ← business outcome
```

## Related decisions

- `docs/decisions/ADR-0001-documentation-strategy.md`
- `docs/decisions/ADR-0002-service-boundary-and-ownership.md`
- `docs/architecture/system-design.md`
