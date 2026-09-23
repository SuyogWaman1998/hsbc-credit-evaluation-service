# ADR-0007: Communication State vs Provider Operation Outcome

- **Status:** Accepted
- **Date:** 2026-09-24
- **Decision owners:** CES design team

## Context

CES communicates with external providers as part of an evaluation. Two different questions must be answered independently:

1. What happened to CES's communication with the provider?
2. What is known about the provider-side business operation?

These are not always the same thing. For example, CES may successfully send a request but receive no response before its timeout. In that case, CES knows the communication attempt timed out, but it may not know whether the provider processed the request.

Using a single state to represent both dimensions would force CES to make assumptions about provider-side processing that it may not be able to prove.

## Decision

CES will represent **communication state** and **provider operation outcome** as separate controlled concepts.

### Communication state

Communication state describes what CES knows about its own interaction with the external provider.

The currently agreed vocabulary is:

```text
NOT_STARTED
CONNECTING
REQUEST_SENT
WAITING_FOR_RESPONSE
RESPONSE_RECEIVED
FAILED
```

These values describe the lifecycle of the communication attempt, not the business outcome of the provider operation.

### Provider operation outcome

Provider operation outcome describes what CES knows about the provider-side operation.

The currently agreed vocabulary is:

```text
UNKNOWN
PROCESSING
COMPLETED
FAILED
NOT_ACCEPTED
```

`UNKNOWN` is a valid and meaningful state. It means CES does not have sufficient evidence to determine the provider-side outcome.

## Example: successful response

```text
communicationState = RESPONSE_RECEIVED
providerOutcome    = COMPLETED
```

CES received a response indicating that the provider operation completed.

## Example: request timeout

```text
communicationState = FAILED
failureType        = READ_TIMEOUT
providerOutcome    = UNKNOWN
```

CES knows that it did not receive the expected response within the configured timeout. It does not automatically know whether the provider processed the request.

## Example: provider explicitly rejects the request

```text
communicationState = RESPONSE_RECEIVED
providerOutcome    = NOT_ACCEPTED
```

The communication itself succeeded, but the provider reported that it did not accept/process the requested operation.

## Example: provider is still processing

```text
communicationState = RESPONSE_RECEIVED
providerOutcome    = PROCESSING
```

The provider responded, but the provider-side operation has not completed yet.

## Why `UNKNOWN` is important

`UNKNOWN` does not mean that CES failed to record what happened.

CES can have complete evidence of its own communication attempt while still lacking evidence of the provider-side business outcome.

For example:

```text
requestSentAt      = 10:01:00
failedAt           = 10:01:30
failureStage       = WAITING_FOR_RESPONSE
failureType        = READ_TIMEOUT
communicationState = FAILED
providerOutcome    = UNKNOWN
```

The record therefore preserves what CES actually observed without inventing a provider-side result.

## State dimensions must not be conflated

The following statements are intentionally different:

```text
Communication failed
```

and:

```text
Provider operation failed
```

The first is a statement about CES's communication. The second is a statement about the provider-side operation and requires sufficient evidence.

Therefore:

```text
Communication failure ≠ Provider failure
```

unless provider evidence establishes the latter.

## Alternatives considered

### Use one combined status

Rejected because a combined state would create ambiguous values and force communication and provider outcomes into one lifecycle. This would make timeout and reconciliation scenarios difficult to represent accurately.

### Treat every timeout as provider failure

Rejected because a timeout only establishes that CES did not receive the expected response within the configured communication window. It does not prove whether the provider received or processed the request.

### Treat every timeout as successful processing

Rejected for the same reason. CES must not assume a successful provider operation without evidence.

## Consequences

### Positive

- CES records observed communication facts separately from provider conclusions.
- Timeout scenarios can be represented without false business outcomes.
- Reconciliation can update the provider outcome later without rewriting communication history.
- Operational and audit investigation becomes clearer.
- Retry decisions can use both dimensions rather than relying on a single ambiguous status.

### Trade-offs

- The interaction model contains multiple state dimensions.
- State combinations must be validated so that impossible or contradictory combinations are not persisted.
- Consumers need to understand the difference between communication state and provider outcome.

## Relationship to failure handling

This ADR intentionally does not finalize retry or reconciliation behavior. Those behaviors will use these two state dimensions as inputs.

The next decision will define how CES handles a communication timeout when the provider-side outcome is unknown.

## Related decisions

- `docs/decisions/ADR-0001-documentation-strategy.md`
- `docs/decisions/ADR-0002-service-boundary-and-ownership.md`
- `docs/decisions/ADR-0003-application-snapshot-and-evaluation-time-data.md`
- `docs/decisions/ADR-0004-evaluation-status-vs-business-decision.md`
- `docs/decisions/ADR-0005-assessment-model.md`
- `docs/decisions/ADR-0006-external-service-interaction-and-evidence.md`
- `docs/architecture/system-design.md`
