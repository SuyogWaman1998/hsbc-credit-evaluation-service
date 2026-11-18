# Credit Evaluation Service — Project Roadmap

**Status:** Agreed  
**Delivery approach:** Incremental delivery  
**Last updated:** 2026-11-18

## 1. Delivery strategy

CES will be developed incrementally, phase by phase. The overall roadmap is established upfront, but detailed design is completed just in time for the phase that needs it.

Each phase follows this engineering cycle:

1. Analyze business requirements, use cases, and failure scenarios.
2. Discuss architecture, alternatives, constraints, and trade-offs.
3. Finalize relevant decisions and record significant decisions in ADRs.
4. Update the system design and related documentation.
5. Implement the finalized scope in the separate CES implementation workspace.
6. Test, review, demonstrate, and sign off the phase.

A phase is not complete merely because its code has been written. Its exit criteria include working behavior, automated tests, reviewed implementation, and documentation that reflects the delivered system.

Implementation may expose a genuine design gap. In that case, revisit the design and record the change through a new or superseding ADR. Do not silently diverge from accepted decisions.

## 2. Project phases

### Phase 1 — Architecture Foundation

**Goal:** Establish the service boundary, domain concepts, and core architectural principles.

**Scope:**
- Service boundary and ownership between Application Service and CES
- Evaluation-time application snapshot
- Evaluation status versus business decision
- Assessment model
- External-service interaction and evidence model
- Communication state versus provider operation outcome
- Timeout, unknown outcome, and reconciliation principles
- Evaluation state machine
- Core domain relationships and outstanding architectural questions

**Current progress:** Partially completed. ADR-0001 through ADR-0008 have been recorded. The evaluation state machine and remaining foundational design details are next.

**Exit criteria:**
- Evaluation lifecycle and allowed transitions are agreed.
- Core domain model and ownership are sufficiently defined to support Phase 2.
- Required persistence concepts and critical invariants are identified.
- System design and relevant ADRs are current.
- Phase 2 implementation scope is agreed.

### Phase 2 — Core Evaluation Engine

**Goal:** Build a working CES foundation that accepts, validates, persists, and processes evaluation requests.

**Scope:**
- REST API contract for creating and retrieving evaluations
- Request validation and error contract
- Evaluation lifecycle implementation
- Core assessment model and domain services
- PostgreSQL persistence and Flyway migrations
- Transaction boundaries and database constraints
- Unit and integration tests

**Exit criteria:**
- A valid request can create an evaluation and persist its evaluation-time data.
- Invalid requests receive consistent validation errors.
- Evaluation lifecycle behavior is covered by automated tests.
- Database schema is migration-managed and reproducible.
- API and persistence behavior match the accepted design.

### Phase 3 — External Service Integration

**Goal:** Integrate CES with external providers required for customer, credit, and fraud assessments.

**Scope:**
- Provider contracts and adapters
- Credit bureau integration
- KYC/customer verification integration
- Fraud service integration
- HTTP client configuration and response mapping
- External interaction evidence and normalized assessment results
- Provider-specific contract tests and simulations

**Exit criteria:**
- CES can invoke the required provider operations through defined adapters.
- Provider responses are mapped into normalized assessment results.
- Communication evidence is persisted according to the agreed model.
- Provider errors and malformed responses are handled and tested.

### Phase 4 — Resilience and Recovery

**Goal:** Ensure failures do not cause incorrect decisions, unsafe duplicate operations, or permanently stuck evaluations.

**Scope:**
- Timeout classification and safe retry rules
- Idempotency for inbound evaluation requests and external operations
- Circuit breaker, bounded retries, and backoff where appropriate
- Unknown provider outcomes
- Provider reconciliation and result retrieval
- Durable recovery scheduling/workers
- Crash recovery and concurrency control

**Exit criteria:**
- Retry decisions respect communication evidence and provider semantics.
- Unknown outcomes are not treated as confirmed provider failures.
- Reconciliation and recovery are durable and testable.
- Duplicate requests and concurrent processing do not create inconsistent business outcomes.

### Phase 5 — Asynchronous Architecture

**Goal:** Introduce event-driven processing where it provides a clear operational or scalability benefit.

**Scope:**
- Kafka topic and event contract design
- Producers and consumers
- Asynchronous assessment workflows
- Consumer idempotency and event ordering considerations
- Retry, dead-letter handling, and replay procedures
- Database/event consistency strategy

**Exit criteria:**
- Agreed asynchronous workflows operate end to end.
- Duplicate and failed message processing is handled safely.
- Event contracts are versioned and documented.
- PostgreSQL remains the authoritative store for CES business state unless a later ADR explicitly changes that decision.

### Phase 6 — Decision Engine

**Goal:** Convert completed assessment evidence into an explainable credit decision.

**Scope:**
- Risk and credit policy rules
- Risk scoring inputs and calculations
- Approval, rejection, and manual-review criteria
- Decision reasons and traceability
- Rule versioning and decision reproducibility
- Boundary and regression tests

**Exit criteria:**
- Decisions are produced only when required assessment inputs and policy conditions are satisfied.
- Decision reasons are traceable to the relevant evidence and rule version.
- Technical processing failures remain distinct from business decisions.
- Decision behavior is covered by representative business scenarios.

### Phase 7 — Production Readiness

**Goal:** Prepare CES for secure deployment and reliable operation.

**Scope:**
- Authentication and authorization
- PII classification, protection, and retention
- Audit requirements
- Structured logs, metrics, tracing, and alerting
- Health and readiness checks
- Docker and Kubernetes deployment
- CI/CD pipeline
- Configuration and secret management
- Capacity, performance, high availability, and disaster recovery

**Exit criteria:**
- Security and data-protection controls are implemented and verified.
- Operators can observe, diagnose, and support the service.
- Deployment is repeatable through the agreed pipeline.
- Performance and operational objectives are measured against defined targets.
- Recovery and deployment procedures are documented and tested.

### Phase 8 — End-to-End System Validation

**Goal:** Validate the complete banking evaluation workflow across normal, exceptional, and recovery scenarios.

**Scope:**
- Application Service to CES integration
- End-to-end evaluation scenarios
- Provider timeout, failure, and delayed-result simulations
- Duplicate requests and concurrent processing
- PostgreSQL and Kafka failure scenarios
- Restart, reconciliation, and recovery scenarios
- Security, performance, and regression validation
- Final architecture-to-implementation review

**Exit criteria:**
- Critical business and failure scenarios pass end-to-end tests.
- No unresolved critical correctness or data-integrity defects remain.
- Architecture documentation reflects the delivered system.
- Known limitations, operational procedures, and future enhancements are documented.

## 3. Phase governance

- Work on one primary phase at a time.
- Do not design every implementation detail of future phases upfront.
- Resolve cross-phase dependencies when they become necessary.
- Keep the architecture roadmap and docs/architecture/system-design.md aligned.
- Use ADRs for significant decisions; preserve history when decisions change.
- A phase can be signed off only when its exit criteria have been demonstrated.
- The phase sequence is a delivery plan, not a claim that every technical activity is strictly sequential. Safe overlap is allowed where dependencies and interfaces are clear.

## 4. Current next step

Continue Phase 1 by defining and reviewing the Credit Evaluation State Machine (planned ADR-0009). Once the lifecycle and core domain invariants are agreed, finalize the Phase 1 exit criteria and begin Phase 2 implementation in the separate CES implementation workspace.
