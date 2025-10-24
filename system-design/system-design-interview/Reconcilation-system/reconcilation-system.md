Question: Build a system to reconcile booking/payments between aggregator and supplier

🧩 Reconciliation System — Architecture Notes

Functional Requirements
- Reconcile data between Agoda and external suppliers.
Detect and report:
  - Missing records 
  - Price mismatches 
  - Duplicate or delayed events
- Support both per-hour and daily reconciliation modes.
- Handle multiple ingestion sources:
  - Supplier pushes events directly to Kafka.
  - Or supplier provides an API, and our ingestion service pushes those events into Kafka.

⸻

Non-Functional Requirements
- High Availability: No missed reconciliations even under node failures.
- Persistence: Full audit trail of all canonical and reconciliation data.
- Scalability: Handle millions of events/day and real-time reconciliation for high-volume suppliers.
- Configurable: Supplier-level policy for reconciliation frequency (hourly or daily).
- Idempotency: Safe re-processing and replay without duplicate anomalies.

⸻

System Flow

1️⃣ Data Ingestion
- Both Agoda and suppliers follow the Outbox Pattern:
- Each write to their system also appends an event to an outbox table.
- Outbox relay publishes to Kafka topics (e.g., agoda.raw.activity, supplier.raw.activity).
- If a supplier cannot push directly, a lightweight ingestion service polls their API and publishes equivalent events into Kafka.

⸻

2️⃣ Canonical Transformer Service
- Consumes raw events from both sides. 
- Normalizes data into a standardized schema:
- activityId, supplierId, activityType, amountMinor, currency, status, version, eventTime, etc.
- Looks up the reconciliation policy (per supplier) to tag each event as:
  - NRT_HOURLY (real-time reconciliation)
  - DAILY_ONLY (batch reconciliation)
- Publishes canonical events to:
  - canonical.agoda.activity and canonical.supplier.activity topics
  - S3 for long-term, append-only persistence (partitioned by date/hour)

⸻

3️⃣ Policy-Based Routing
- A lightweight Router service subscribes to canonical topics.
- Based on supplier policy:
- Routes NRT_HOURLY events to real-time reconciliation topics 
- Routes DAILY_ONLY events to batch storage only 
- All events, regardless of mode, are appended to S3 for audit and daily batch processing.

⸻

4️⃣ Real-Time / Hourly Reconciliation
- A Reconciliation Service consumes events from canonical.*.nrt topics (Agoda + Supplier). 
- Keeps short-term state to match records:
  - Option A: Redis (with TTL = grace window)
  - Option B: In-memory store for last 1 hour (faster but volatile)
- On event arrival:
  - If counterpart exists → compare using defined rules (price, timestamp skew, duplicates, etc.)
  - If counterpart missing → store record and set expiry/grace timer.
- On timer expiry → mark as MISSING_SUPPLIER / MISSING_AGODA anomaly and emit event.
- Comparison results and anomalies are pushed to:
  - reco.results.nrt topic for downstream alerting
  - S3 / ClickHouse / Data Warehouse for analytics

⸻

5️⃣ Daily Reconciliation (Batch Job)
- A daily batch job (e.g., Airflow/Spark/Athena) reads canonical data from S3.
- Process:
  - Load Agoda + Supplier records for the day.
  - Deduplicate and select latest version per activityId.
  - Perform full outer join on activityId.
  - Apply same reconciliation rules (price tolerance, FX conversion, time skew, etc.)
  - Writes results to:
  - reconciliation_facts/ and anomalies/ folders in S3.
  - These are the authoritative daily results for reporting and audits.

⸻

6️⃣ Anomaly Detection & Alerting
- Anomalies include:
  - MISSING_SUPPLIER 
  - MISSING_AGODA 
  - PRICE_MISMATCH 
  - DUPLICATE_*
  - TIMESTAMP_SKEW 
  - Anomalies are streamed to an Alerting Service that groups by supplier and severity. 
  - Alerts sent via Slack / Email / PagerDuty / Dashboard.

📊 Reconciliation Rules
1.	Existence Check: Missing record on either side.
2.	Price Check:
```
abs(agodaPriceUSD - supplierPriceUSD) > max(absTol, pctTol * agodaPriceUSD)
```
3.	Duplicate Check: Same activityId seen multiple times.
4. Timestamp Skew: Event time difference > allowed skew.
5. Status Alignment: BOOKED vs REFUNDED mismatch.

⚙️ Operational Notes
- Outbox pattern ensures no event loss between DB and Kafka. 
- Idempotent processing: (source, activityId, version) key prevents duplicates. 
- Audit trail: S3 is append-only; daily runs are re-playable. 
- Config-driven behavior: change reconciliation mode or thresholds without redeploy. 
- Monitoring: unmatched count, Redis state size, reconciliation lag, anomaly rate.
