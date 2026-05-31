# Justification — Real-time Fraud Scoring (Scenario A)

## Serving pattern: Online (synchronous, in-path)

The scenario states two hard constraints: scores must be delivered before the user sees a payment confirmation, and p95 latency must stay under 80 ms end-to-end. Both constraints eliminate batch. A batch pattern delivers predictions ahead of time; it cannot score a transaction that has not happened yet. A pure streaming pipeline (event-driven, async) could achieve high throughput but introduces non-deterministic delay — the score might arrive after the confirmation screen has already rendered. The only pattern that satisfies "before confirmation" with a bounded latency SLA is synchronous online serving: the payment gateway blocks on the fraud decision before returning a response to the app.

The 300 tx/s peak throughput is comfortably within the range of a horizontally scalable inference service behind a load balancer. A single GPU instance can score well above 1 000 requests/second for a model of the complexity typical of fraud detection (gradient-boosted tree or a shallow neural network on tabular features). Horizontal scaling handles burst without architectural change.

## Where inference runs: Cloud, regionally deployed

Inference runs in the cloud, not at the edge. The model requires two feature classes — account history (30-day velocity, prior fraud flags) and device fingerprint (risk score derived from session signals) — both of which are maintained in a centralised online feature store. Pushing inference to the edge would require either (a) syncing sensitive account data to the device, which is a security and compliance problem, or (b) making a network call back to the feature store, which re-introduces the latency that edge deployment was supposed to remove. A cloud-hosted scoring service with a regional deployment (co-located with the payment gateway's region) keeps the feature lookup on a low-latency internal network path and avoids public-internet round-trips.

## Targets: optimise latency and throughput; cost is the budget

**Latency target:** p95 ≤ 80 ms end-to-end. The user is staring at a payment confirmation screen; any delay beyond ~200 ms is perceptible and erodes trust. 80 ms is the stated SLA — the scoring service alone must budget ≤ 30 ms (leaving headroom for gateway overhead, feature store lookup, and decision routing).

**Throughput target:** sustain 300 tx/s at peak with no queue build-up. The service must auto-scale ahead of peak (e.g., pre-warm replicas before market-hours traffic ramps).

**Cost (budget constraint):** accept higher infrastructure spend — multiple inference replicas, a low-latency Redis cluster — to meet the latency and throughput targets. Fraud losses from a slow or unavailable scorer dwarf the infra cost. Cost optimisation is secondary; it applies only to the offline training pipeline, where batch compute is cheap.

## Fallback when the model is unavailable or wrong

**Scoring service down:** a circuit breaker in the Decision Engine detects consecutive failures and switches to a rule-based fallback (velocity checks on amount and merchant category only). The fallback routes borderline transactions to `STEP_UP_AUTH` rather than `BLOCK`, preserving conversion while adding friction. Every fallback decision is flagged in the Data Lake for post-hoc review.

**Feature store unavailable:** score with degraded features (transaction context only, no account history). The model outputs a higher-uncertainty score; the Decision Engine lowers the `ALLOW` threshold, routing more traffic to `STEP_UP_AUTH` until the feature store recovers.

**Model wrong (false positives / false negatives):** the Monitoring & Drift Detector tracks precision/recall against confirmed fraud labels from the Label Store (chargebacks, ops investigations). When recall drops below the agreed floor or PSI on score distribution exceeds the threshold, an alert triggers the Training Pipeline to retrain. The Model Registry maintains the last-known-good version for instant rollback.
