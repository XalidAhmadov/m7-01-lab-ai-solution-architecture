# ADR 0001: Pre-materialise features into an online feature store rather than querying the OLTP database at inference time

## Context

The fraud scoring service needs account history (spend velocity, prior fraud flags) and device fingerprint for every transaction. These features live in the transactional database, but the p95 latency budget is 80 ms end-to-end, leaving the feature lookup roughly 10–15 ms. A direct query against the OLTP database under 300 tx/s peak will saturate connection pools and push p99 latency well above budget.

## Decision

We pre-materialise the required features into a dedicated online feature store (Redis-backed, served via Feast). A nightly batch job (and a lightweight streaming job for high-churn features like spend velocity) writes pre-computed feature vectors keyed by account ID and device ID. The scoring service performs a single key-value lookup — typically 1–3 ms — rather than a multi-table SQL join.

## Alternatives rejected

- **Query OLTP at inference time:** simple to implement but does not survive 300 tx/s peak — connection pool exhaustion and p99 spikes disqualify it. Also couples the fraud service to the payments database schema.
- **Embed all feature computation inside the scoring service (stateless, request-only features):** possible for device signals derived from the request payload, but account history cannot be recomputed from the transaction alone; it requires historical aggregates, which must come from somewhere persistent.
- **Stream-compute features in real time (Flink / Kafka Streams):** would provide fresher velocity features but adds significant operational complexity and a new failure mode in the critical path. The marginal freshness gain (minutes vs. hours for spend velocity) does not justify the risk given the weekly churn rate of fraud patterns.

## Consequences

- The online feature store becomes a critical dependency of the scoring service; its availability SLA must match or exceed the payment gateway's.
- Feature freshness is bounded by the batch cadence (nightly for most features, near-real-time for spend velocity via a streaming job). A sudden account compromise between refresh windows may be scored on stale features.
- Feature engineering logic is duplicated: once in the training pipeline (to produce training labels) and once in the materialisation job. The Feast feature definitions must be kept in sync to avoid training–serving skew.

## Revisit if

The fraud pattern shifts to require sub-minute feature freshness (e.g., card-testing attacks that complete within a single batch window), at which point a full streaming feature pipeline becomes worth the operational cost.
