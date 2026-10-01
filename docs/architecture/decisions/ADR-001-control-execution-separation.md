# ADR-001: Separate Control and Execution Planes

## Decision
The research orchestrator and API form the trusted control plane. Web, browser, processing, simulation, ML and reporting workloads run as separately permissioned execution workers.

## Rationale
Research inputs include hostile or untrusted content. A scraped page or model output must not be able to mutate workflow policy, access secrets or cross tenant boundaries.

## Consequences
- More services and schemas.
- Clearer security boundaries.
- Independent scaling.
- Safer retries and failure recovery.
