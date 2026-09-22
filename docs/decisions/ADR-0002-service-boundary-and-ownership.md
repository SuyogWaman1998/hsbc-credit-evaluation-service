# ADR-0002: CES Service Boundary and Ownership

- **Status:** Accepted
- **Date:** 2026-09-23
- **Decision owners:** CES design team

## Context

The loan application lifecycle and the credit evaluation lifecycle are related but represent different business responsibilities. Multiple channels or actors may create or initiate a loan application, but the application itself is owned by the Application Service. Credit evaluation is a separate capability that must evaluate the application and maintain its own evaluation lifecycle and audit trail.

We need a clear service boundary so that CES does not become the owner of application data or expose internal evaluation orchestration to callers.

## Decision

The Application Service owns the live loan application lifecycle. The Credit Evaluation Service (CES) owns the credit evaluation lifecycle.

The normal service interaction is:

```text
Customer / Channel
        ↓
Application Service
        ↓
POST /credit-evaluations
        ↓
Credit Evaluation Service
```

The Application Service sends the application information required for evaluation as part of the evaluation request. CES creates and manages an independent evaluation using that information.

CES owns:

- Evaluation identity and lifecycle.
- Evaluation status.
- Evaluation-time application snapshot required for the evaluation.
- Customer validation assessment.
- Credit assessment.
- Fraud assessment.
- Evaluation decision and decision reason when a business decision is reached.
- Evidence of relevant external-service interactions used by the evaluation.

The Application Service continues to own:

- The live loan application.
- The application lifecycle outside the evaluation process.
- Application changes that occur independently of CES.

CES is therefore not the system of record for the live application.

## Boundary principle

CES should expose the evaluation capability through its service boundary rather than requiring callers to know the internal orchestration between assessments, external providers, persistence, and other components.

Conceptually:

```text
                 Application Service
                         │
                         │ evaluation request
                         ▼
              ┌─────────────────────┐
              │ Credit Evaluation   │
              │ Service             │
              ├─────────────────────┤
              │ Evaluation          │
              │ Assessments         │
              │ Decision            │
              │ External evidence   │
              └─────────────────────┘
```

## Alternatives considered

### Application Service owns evaluation logic

Rejected because it would couple the application lifecycle to credit-evaluation orchestration and make the evaluation capability harder to evolve independently.

### CES owns the complete loan application

Rejected because the live application belongs to the application domain. Duplicating ownership in CES would create competing sources of truth.

### External channels call CES directly for evaluation

Not selected as the primary ownership model. Channels should interact with the application capability that owns the loan application; that application capability then invokes CES as part of the application processing flow.

## Consequences

### Positive

- Clear ownership of application and evaluation lifecycles.
- CES can evolve its evaluation orchestration independently.
- The live application remains authoritative in the Application Service.
- CES can preserve the evaluation-time data required for audit and reproducibility.
- Callers do not need to understand CES internal orchestration.

### Trade-offs

- Some application data is represented in CES as an evaluation-time snapshot.
- Cross-service consistency must be considered when application and evaluation state change independently.
- The API contract between the Application Service and CES becomes an important integration boundary.

## Related design

The evaluation-time application snapshot is documented in the system design and will be captured in a dedicated ADR when that decision is formally recorded.

See also:

- `docs/decisions/ADR-0001-documentation-strategy.md`
- `docs/architecture/system-design.md`
