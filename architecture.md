# Architecture — Real-time Fraud Scoring (Scenario A)

```mermaid
flowchart LR
    subgraph online ["Online path  (p95 ≤ 80 ms)"]
        PG[Payment Gateway]
        ES[Event Stream\nKafka / Kinesis]
        FSvc[Fraud Scoring\nService]
        FS[Online Feature Store\nRedis / Feast]
        DE[Decision Engine]
        APP[Payment App]
    end

    subgraph offline ["Offline / Batch"]
        DL[Data Lake\nS3 / BigQuery]
        LS[Label Store\nchargebacks + ops]
        TP[Training Pipeline\nSpark / SageMaker]
        MR[Model Registry\nMLflow]
    end

    subgraph monitoring ["Monitoring"]
        MON[Monitoring &\nDrift Detector]
    end

    %% Online flow
    PG -->|transaction event| ES
    ES -->|enriched event| FSvc
    FS -->|account history\ndevice fingerprint| FSvc
    MR -->|model artifact\non deploy| FSvc
    FSvc -->|risk score + features| DE
    DE -->|block / allow /\nstep-up auth| APP

    %% Write paths
    ES -->|raw events| DL
    DE -->|decisions + scores| DL

    %% Offline flow
    DL -->|labeled transactions| TP
    LS -->|fraud labels| TP
    TP -->|new model version| MR
    TP -->|batch feature write| FS

    %% Feedback loop
    FSvc -->|latency + score dist| MON
    DE -->|outcome signals| MON
    MON -->|drift alert /\nperformance decay| TP
```

## Serving boundary

| Zone | Components |
|------|-----------|
| **Online** (synchronous, in payment path) | Payment Gateway → Event Stream → Fraud Scoring Service → Decision Engine → Payment App |
| **Online read** (sub-ms lookup) | Online Feature Store (serves account history & device fingerprint to the scoring service) |
| **Offline / Batch** | Data Lake, Label Store, Training Pipeline, Model Registry |
| **Async** | Monitoring & Drift Detector (reads scoring output out-of-band) |

## Key contracts

| Arrow | What flows |
|-------|-----------|
| PG → Event Stream | Transaction event (amount, merchant, timestamp, device ID, account ID) |
| Event Stream → Fraud Scoring Service | Enriched event (same + session metadata) |
| Online Feature Store → Fraud Scoring Service | Pre-materialized features (30-day spend velocity, device risk score, account age) |
| Fraud Scoring Service → Decision Engine | Risk score (0–1), feature vector, model version |
| Decision Engine → Payment App | Decision enum: `BLOCK` / `ALLOW` / `STEP_UP_AUTH` |
| Training Pipeline → Online Feature Store | Batch feature refresh (nightly) |
| Monitoring → Training Pipeline | Drift signal / performance-decay trigger |
```
