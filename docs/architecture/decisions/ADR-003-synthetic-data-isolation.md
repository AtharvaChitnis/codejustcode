# ADR-003: Keep Synthetic Data Explicitly Typed

## Decision
Synthetic respondents/responses are represented as simulated artefacts and are never silently merged into observed research data.

## Rationale
A large synthetic sample is not evidence that the real population responded. Explicit provenance is needed for interpretation, reporting and validation.

## Consequences
Dashboard/report schemas need a data-class field and publication checks must reject unlabeled synthetic outputs.
