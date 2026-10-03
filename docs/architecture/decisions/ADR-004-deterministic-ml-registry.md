# ADR-004: Approved ML Registry

## Decision
Production ML execution uses a fixed registry of supported algorithms and versioned training wrappers. The agent selects from the registry but does not generate arbitrary training code at runtime.

## Rationale
Determinism, security, reproducibility and easier validation.

## Consequences
Adding a new algorithm requires a reviewed implementation and updated evaluation tests.
