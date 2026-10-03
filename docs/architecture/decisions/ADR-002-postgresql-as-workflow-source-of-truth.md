# ADR-002: PostgreSQL Is the Workflow Source of Truth

## Decision
Persist research job state, ownership, checkpoints, artifact metadata and workflow transitions in PostgreSQL. Redis is used for queues, locks and caching.

## Rationale
Queue state can disappear or become inconsistent during failure. The workflow must be reconstructible from durable state.

## Consequences
The orchestrator must reconcile persisted state and queue state and workers must be idempotent.
