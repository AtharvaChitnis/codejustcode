# AI Market Research Agent — Production Architecture

## Status
Production reference architecture and implementation blueprint.

## Product mission
Automate market research from a natural-language request through evidence collection, dataset discovery, scraping, synthetic respondent simulation, predictive/statistical analysis, validation, and evidence-linked reporting.

Research integrity invariant: observed, inferred, simulated, and predicted artefacts remain explicitly typed and traceable.

## Architecture principles
- Agentic control plane; deterministic execution plane.
- PostgreSQL is authoritative workflow/state storage.
- Redis is queue/cache/coordination, never the sole workflow source of truth.
- Every stage emits typed artefacts with provenance.
- Workers are least-privileged, bounded, idempotent, and retry-safe.
- Synthetic respondents are simulations, not substitutes for observed respondents.
- Narrative generation can only cite approved evidence/model artefact IDs.
- External web/model providers are untrusted dependencies.
- Expensive workloads scale independently on Kubernetes.
- Budget, quality, security, and validation gates are first-class workflow controls.

## System context

```text
User / Researcher
      |
      v
Next.js Research Dashboard
      |
      v
FastAPI API Gateway
      |
      +---- PostgreSQL  <---- metadata / state / ownership / lineage
      |
      +---- Redis       <---- queues / locks / cache / quotas
      |
      v
Research Orchestrator
      |
      +--> Web Research Workers ------> Search / Web
      +--> Scraper Workers -----------> External Web
      +--> Dataset Discovery ----------> Public Dataset Sources
      +--> Processing Workers
      +--> Synthetic Simulation Workers
      +--> ML / Statistics Workers
      +--> Validation Workers
      +--> Reporting Workers
      |
      +----> Object Storage ------------> raw / processed / models / reports
      +----> LLM / Embedding Providers
```

## Control, execution and persistence planes

### Control plane
FastAPI, authentication/authorization, policy enforcement, workflow state, budgets, quotas, publication controls.

### Execution plane
Short-lived workers for browsing, retrieval, extraction, ETL, simulation, ML, validation and reporting.

### Persistence plane
- PostgreSQL: system of record.
- Redis: transient coordination and queues.
- Object storage: immutable large artefacts.
- Telemetry platform: logs, metrics and traces.

No untrusted page content, LLM output, or worker result directly mutates authoritative state. All mutations pass through validated schemas and policy-aware services.

## End-to-end workflow

```text
CREATED
  -> PLANNING
  -> COLLECTING
  -> PROCESSING
  -> POPULATION_BUILD
  -> SIMULATING
  -> ANALYZING
  -> VALIDATING
  -> PUBLISHING
  -> COMPLETED

Retryable failure -> RETRYING -> previous state
Weak validation  -> RECALIBRATE / REPLAN
Budget/policy    -> BLOCKED
Cancellation     -> CANCELLING -> CANCELLED
Terminal failure -> FAILED
```

### Stage responsibilities
1. Intake — normalize objective, audience, geography, questions, constraints and output requirements.
2. Planning — generate hypotheses, evidence plan, data requirements, model candidates, stop conditions and budget.
3. Evidence collection — search, retrieve, scrape and discover datasets in parallel where safe.
4. Processing — schema validation, cleaning, deduplication, entity resolution and feature engineering.
5. Population synthesis — create a versioned synthetic population from approved reference distributions.
6. Response simulation — produce question-conditioned probability distributions and sample responses.
7. Analysis — train approved models, run segmentation, threshold optimization and scenarios.
8. Validation — calibration, holdout evaluation, leakage checks, distribution checks and uncertainty analysis.
9. Insight generation — produce structured findings linked to evidence and model artefacts.
10. Publishing — report, charts, exports and dashboard configuration.

## Specialized agents

### Research Planner
Input: research request.
Output: typed ResearchPlan.
Controls: allowed tools, max calls, budget, deadlines, data policy.

### Web Research Agent
Flow: question -> query expansion -> search provider -> candidate URLs -> retrieval -> extraction -> source classification -> claim extraction -> EvidenceClaim.
Never returns a narrative as authoritative output.

### Scraping Agent
- Playwright/Selenium for JS-heavy pages.
- requests/BeautifulSoup for simple pages.
- Source-specific adapters for high-value sites.
- Raw snapshot + normalized content.
- Per-domain concurrency/backoff.
- robots/access/ToS controls.
- response-size and download limits.
- schema validation failures are explicit outcomes.

### Dataset Discovery Agent
Creates candidate inventory: source, schema, coverage, geography, target, missingness, license, provenance, relevance. Only approved versions enter feature pipelines.

### Synthetic Respondent Engine
Represent respondent attributes explicitly and version all generation configuration.

```text
Reference distributions
       +
Historical/public behavioural data
       +
Research context
       |
       v
Population synthesis
       |
       v
Behavioural model
       |
       v
Question-conditioned distribution
       |
       v
Probabilistic sampling
       |
       v
Synthetic response batch
```

For structured questions, the probability model chooses the behaviour. The LLM is only used to realize constrained open-text explanations after the structured response has been fixed.

### ML Analysis Engine
Approved registry:
- Logistic Regression
- Random Forest
- XGBoost
- Linear/Ridge regression
- K-Means
- HDBSCAN/hierarchical clustering
- PCA
- UMAP for visualization

Training flow:
raw -> schema validation -> missingness -> encoding/transforms -> leakage checks -> feature selection -> split -> preprocessing artifact -> train -> evaluate -> register.

The model agent selects from the registry; production never executes arbitrary generated model code.

### Validation Layer
Hard gates:
- out-of-sample performance,
- probability calibration,
- leakage checks,
- synthetic-vs-reference distribution comparison,
- uncertainty estimation,
- schema/data contract validation.

### Insight / Report layer
Narrative output is generated from structured findings only. Every externally visible claim must link to an EvidenceClaim, ModelRun/ValidationRun, or explicit SimulationBatch.

## Synthetic-response governance

Four epistemic classes are mandatory:

| Class | Meaning | Use |
|---|---|---|
| Observed | retrieved/user-provided evidence | evidence, statistics, licensed modeling |
| Inferred | derived from observed data | analysis with lineage |
| Simulated | generated respondents/responses | scenarios, prototyping, sensitivity analysis |
| Predicted | validated model output | forecasts/scenarios with uncertainty |

Synthetic sample size is not a real-world sample size.
Every synthetic batch stores simulation_batch_id, generation_config_id, model_version, seed lineage, reference data versions, and calibration_report_id.

Calibration loop:
```text
generate -> summarize -> compare_to_reference
                 |
          error > tolerance
                 |
              recalibrate
                 |
              generate
```

If tolerance is not achieved, the batch is marked unvalidated and cannot silently become observed evidence.

## Data architecture

### PostgreSQL
Core entities: tenant, user, project, research_job, research_plan, source, evidence_claim, dataset, dataset_version, processing_run, survey_question, simulation_batch, synthetic_respondent, response_batch, model_run, model_artifact, validation_run, insight, report, artifact, audit_event.

Use tenant-scoped foreign keys and indexes for every query path.

### Redis
Queues:
- research:plan
- research:web
- research:scrape
- research:dataset
- research:process
- research:simulate
- research:ml
- research:validate
- research:report

Each queue has concurrency limit, retry policy, visibility timeout, DLQ, priority and tenant quotas.

### Object storage
Recommended prefixes:
```text
/{tenant_id}/{project_id}/{research_id}/
  raw/
  processed/
  simulations/
  models/
  validation/
  reports/
  manifests/
```
Objects are content-addressed or generation-versioned.

## Provenance and reproducibility
Every major artefact includes artifact_id, content_hash, schema_version, producer_version, created_at, research_id, job_version, and upstream_artifact_ids.

Every research run creates an immutable manifest containing configuration, prompts/templates, model IDs, dataset versions, tool/provider versions, container/image version, seeds, and policy version.

A report is publishable only when every citation resolves to an approved provenance record.

## API surface
Versioned REST:
- POST /api/v1/research
- GET /api/v1/research/{id}
- POST /api/v1/research/{id}/run
- POST /api/v1/research/{id}/cancel
- GET /api/v1/research/{id}/sources
- GET /api/v1/research/{id}/datasets
- GET /api/v1/research/{id}/simulation
- GET /api/v1/research/{id}/models
- GET /api/v1/research/{id}/validation
- GET /api/v1/research/{id}/report
- GET /api/v1/research/{id}/dashboard

Create/run operations require Idempotency-Key.

Error envelope:
```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "Human-readable summary",
    "request_id": "req_123",
    "details": []
  }
}
```

## Event contract
Event types:
research.created, research.plan.ready, data.collection.requested, data.collection.completed, data.processing.completed, simulation.population.ready, simulation.batch.completed, model.run.requested, model.run.completed, validation.completed, insight.generated, report.generated, research.completed, research.failed.

Event envelope includes event_id, event_type, event_version, tenant_id, research_id, job_version, produced_artifact_ids, occurred_at, producer_version, and attempt. Consumers must be idempotent.

## Task envelope and recovery
A task envelope contains research_id, job_version, task_id, task_type, attempt, idempotency_key, tenant_id, input_artifact_ids, policy_version, and deadline.

Production rules:
- unique logical task keys,
- content hashes for artefacts,
- exponential backoff + jitter,
- explicit retryable/terminal/policy-blocked errors,
- checkpoints before state advancement,
- dead-letter queues,
- reconciliation controller for DB/queue divergence,
- durable cancellation requests,
- bounded tool and stage timeouts.

## Kubernetes topology
Namespaces:
market-api, market-workers, market-scrapers, market-ml, market-observability.

Worker pools:
- API: stateless, HPA.
- Agent/research: queue-depth scaling.
- Scraper: isolated nodes, browser memory/CPU limits, queue-depth scaling.
- ML: CPU pool; optional GPU pool.
- Reporting: burst CPU pool.
- Scheduler/orchestrator: small fixed replica set with leader/lock semantics.

Primitives: Deployments, Jobs, KEDA/HPA-style autoscaling, Secrets, ConfigMaps, NetworkPolicies, PodDisruptionBudgets, resource requests/limits and topology constraints.

Do not mount host filesystem, Docker socket, cluster-admin tokens, or cloud metadata endpoints into scraper/browser pods.

## Security model
Threats include prompt injection, SSRF, malicious downloads, credential exfiltration, cross-tenant access, unsafe browser execution, runaway agents, and dependency compromise.

Controls:
- treat web content as data, never instructions;
- egress allowlists;
- deny RFC1918/link-local/metadata endpoints unless explicitly allowed;
- MIME/size/archive scanning;
- sandbox file parsing;
- scoped service credentials;
- secret redaction;
- server-side tenant authorization;
- token/tool/call/wall-clock/spend budgets;
- pinned dependencies and image scanning;
- signed/approved base images;
- security events for blocked actions.

## Multi-tenancy
Every tenant-owned entity includes tenant_id. Authorization is enforced server-side. Object storage paths and queue messages carry tenant scope. Per-tenant controls include concurrency quotas, spend limits, priority, retention policy, and audit trail.

Browser never receives storage credentials; use short-lived signed URLs.

## Observability
Metrics: job_duration_seconds, queue_depth, scrape_success_rate, model_auc, pr_auc, calibration_error, validation_pass, source_coverage, duplicate_rate, missingness, llm_calls, browser_minutes, storage_bytes, cpu_seconds.

Every trace should correlate request_id -> research_id -> task_id -> worker_id -> artifact_id.

Alerts should cover queue starvation, retry storms, validation regressions, budget overruns, database saturation, worker crash loops, and provider error spikes.

## Cost controls
Budget fields: max_wall_clock, max_llm_spend, max_browser_minutes, max_simulation_rows, max_embedding_tokens.

Controls:
- cache retrieval and schema inspection,
- batch embeddings/simulation/scoring,
- small models for extraction/classification,
- stronger models for planning/ambiguity,
- early stop dominated models,
- adaptive simulation sample sizes,
- queue backpressure,
- stage-level budgets.

## Repository layout
```text
market-research-agent/
├── apps/
│   ├── web/
│   └── api/
├── services/
│   ├── orchestrator/
│   ├── web-research/
│   ├── scraper/
│   ├── dataset-discovery/
│   ├── processing/
│   ├── simulation/
│   ├── ml/
│   ├── validation/
│   └── reporting/
├── packages/
│   ├── schemas/
│   ├── db/
│   ├── queue/
│   ├── observability/
│   └── common/
├── infra/
│   ├── docker/
│   ├── k8s/
│   └── terraform/
├── data/
│   └── examples/
├── tests/
└── docs/
    ├── architecture/
    ├── research-methods/
    └── api/
```

## Implementation phases
1. FastAPI + PostgreSQL + orchestrator state machine + typed schemas.
2. Local CSV/dataset ingestion + deterministic processing pipeline.
3. Synthetic population + probabilistic survey engine + calibration benchmark.
4. ML registry + baseline/candidate models + evaluation + threshold/scenario engine.
5. Web research + scraping + dataset discovery + evidence registry.
6. Validation + provenance + insight/report generation.
7. Next.js dashboard.
8. Kubernetes hardening, autoscaling, security, observability, backups and disaster recovery.

Build-order rule: prove calibration and real-data holdout performance before increasing synthetic population scale or optimizing Kubernetes throughput.

## Failure modes

| Failure | Detection | Response |
|---|---|---|
| weak search | coverage/quality | query re-plan |
| scraper break | schema failures | adapter/fallback parser |
| synthetic drift | calibration error | recalibrate or mark unvalidated |
| synthetic looks plausible but fails | real holdout benchmark | block publication |
| class imbalance | PR-AUC/per-class metrics | threshold/cost-sensitive evaluation |
| leakage | feature lineage/split checks | fail validation |
| fabricated citation | provenance validation | block report |
| worker crash | heartbeat/timeout | checkpoint + retry |
| cost spike | budget counters | hard stop/backpressure |
| stale dashboard | artifact version mismatch | invalidate stale state |

## Production acceptance criteria
- research jobs have durable state and resumability;
- every major stage is idempotent;
- data collection returns provenance-backed records;
- synthetic batches are explicitly labelled and versioned;
- calibration precedes publication;
- ML runs retain data/split/preprocessing/model metadata;
- validation can block publication;
- every narrative claim resolves to structured evidence or validated model artefacts;
- API resources enforce tenant authorization;
- queue workers have retry/DLQ/cancellation semantics;
- Kubernetes workloads scale independently;
- logs/metrics/traces correlate across a research run;
- backups and recovery procedures have been exercised.

## Recommended baseline stack
- Next.js + TypeScript
- FastAPI + Python
- PostgreSQL
- Redis
- S3-compatible object storage
- Python worker services
- scikit-learn + XGBoost
- Playwright/Selenium + requests/BeautifulSoup
- LLM provider abstraction with hosted APIs or local Ollama
- Docker for local development
- Kubernetes for production

## Key engineering decision
Do not make the LLM the database, workflow engine, statistical model, or security boundary.

Use the LLM for:
- understanding ambiguous research requests,
- proposing plans/hypotheses,
- selecting approved tools/models,
- summarizing validated artefacts,
- constrained qualitative text.

Use deterministic services for:
- state,
- persistence,
- scraping,
- ETL,
- statistical modelling,
- validation,
- authorization,
- audit and publication.
