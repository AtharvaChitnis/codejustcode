# AI Market Research Agent — Architecture Starter

This repository contains the production architecture reference for an AI market-research platform.

## Design goal
Automate:
research request -> web research -> scraping -> dataset discovery -> processing -> synthetic respondents -> ML/statistics -> validation -> evidence-linked report -> dashboard.

See AI_MARKET_RESEARCH_ARCHITECTURE.md for the full design.

## Suggested first implementation
1. FastAPI API
2. PostgreSQL state and metadata
3. Redis task queues/cache
4. Python workers
5. Synthetic simulation + calibration benchmark
6. ML baseline + validation
7. Web research/scraping
8. Next.js dashboard
9. Kubernetes

## Important
Synthetic respondents are simulations, not observed survey participants. Preserve provenance and validate synthetic behaviour against appropriate observed reference data before using it for consequential conclusions.

## Existing repository
The existing p/ directory is a small Express/React exercise and contains checked-in node_modules. The new architecture is intentionally isolated from it.
