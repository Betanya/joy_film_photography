# Google Data Engineer — One‑Day Intensive Study Guide + Mock Exam

This plan is optimized for one day of focused prep, aligned to the four domains: Cloud Data Architecture & Modeling, Data Processing & Performance Tuning, Data Governance/Security, and Practical SQL.

---

## One‑Day Schedule (8–10 hours)

- 0:00–0:30: Warm‑up & goals
  - Review domains, set priorities based on your strengths.
- 0:30–2:00: Cloud Architecture (GCP Lake/Lakehouse)
  - Services mapping, patterns, ingestion, storage formats, catalog/governance.
- 2:00–3:30: Processing & Performance (Spark + BigQuery)
  - Shuffles, partitioning, AQE, joins, file sizes; BigQuery tuning.
- 3:30–4:30: Security/Governance
  - IAM, RBAC/ABAC, column masking, DLP, CMEK, VPC‑SC, auditing.
- 4:30–6:00: SQL Mastery
  - Windows, CTEs, advanced joins, BigQuery array/struct basics.
- 6:00–8:00: Full Mock Exam (timed)
  - 90 minutes to answer, 30 minutes to self‑grade.
- 8:00–8:30: Review & polish
  - Summarize gaps, rehearse narratives, system design trade‑offs.

Tips:
- Use a timer; stop when timebox ends.
- Keep a paper checklist of trade‑offs: latency, cost, reliability, complexity, ops.

---

## Core Reference — GCP Mapping

- Storage: Cloud Storage (data lake), BigQuery (lakehouse warehouse), Bigtable (low‑latency wide‑column), Spanner (global RDBMS), Firestore.
- Compute: Dataflow (Beam, streaming/batch), Dataproc (Spark/Hadoop), BigQuery (SQL engine), Cloud Functions/Run (serverless glue), Vertex AI (ML).
- Ingestion: Pub/Sub (events), Transfer Service (cloud→GCS), Storage Transfer (on‑prem→GCS), CDC via Datastream.
- Catalog/Governance: Dataplex (domains, lakes, assets), Data Catalog (tags), DLP, IAM Conditions (ABAC), Cloud KMS (CMEK).
- Orchestration: Cloud Composer (Airflow), Workflows.

---

## Domain 1: Cloud Data Architecture & Modeling

What good looks like:
- Clear separation of zones: raw (landing) → curated (conformed) → serving (warehouse/marts).
- Event streams (Pub/Sub + Dataflow) complement batch (Transfer/Spark/Beam).
- Columnar + compressed formats (Parquet/ORC/Avro), partitioning by time or entity.
- Strong catalog and governance via Dataplex + Data Catalog tags.
- DR/HA with multi‑region buckets and BigQuery dataset replication.

High‑yield patterns:
- Lakehouse on GCP: Cloud Storage for raw/curated Parquet; BigQuery for serving marts; Dataplex to govern lakes and assets; Dataflow for streaming ETL; Dataproc for Spark batch; Pub/Sub for ingestion.
- Schema evolution: Avro or Parquet with explicit schema management; use compatible changes; late‑binding views in BigQuery.
- Real‑time vs batch: Use Pub/Sub + Dataflow for sub‑second ingest; batch compaction via Dataproc for small files; landing→curation SCD with merge in BigQuery.

Trade‑offs to articulate:
- Latency vs cost: Streaming (Dataflow) more expensive than batch but enables fraud detection; BigQuery slots vs on‑demand.
- Complexity vs reliability: Presto/Spark on Dataproc flexible but operational overhead; BigQuery serverless simpler but limits custom logic.
- Storage vs query efficiency: GCS cheaper/agnostic; BigQuery faster analytics; dual‑write for balance.

Design template (Fraud detection, millions of users):
- Ingestion: Mobile/web events → Pub/Sub; CDC of transactions via Datastream.
- Stream processing: Dataflow pipeline performs sessionization, joins with reference data, risk scoring.
- Storage: Raw events in GCS (Parquet); curated features in BigQuery; hot feature store in Bigtable or Redis (via Memorystore).
- Serving: Real‑time dashboards in Looker Studio; REST scoring via Cloud Run.
- Governance: Dataplex domains; Data Catalog tags for sensitivity; CMEK via Cloud KMS.
- Reliability: Regional + multi‑region topics; Dataflow autoscaling; dead‑letter topics; replay from Pub/Sub retention.

Checklist:
- Zones (raw/curated/serving) defined?
- Formats (Parquet/Avro) + partitioning plan?
- Streaming/backfill story?
- Catalog/tags + data lineage?
- DR/HA, replay, idempotency?

---

## Domain 2: Data Processing & Performance Tuning

Spark (Dataproc) essentials:
- Partitioning: `repartition()` for parallelism; `coalesce()` to reduce tasks; align to output partition columns.
- File sizing: Target 128–512MB per file for Parquet; avoid millions of tiny files.
- Shuffles: Minimize wide transformations; pre‑aggregate; `mapPartitions`; combine with `reduceByKey` instead of `groupByKey`.
- Joins: Broadcast small dimension (`broadcast()`); sort‑merge join on partitioned/sorted data.
- Skew: Salting keys; `spark.sql.autoBroadcastJoinThreshold`; AQE (`spark.sql.adaptive.enabled=true`), skew join handling.
- Caching: Persist reused DataFrames; unpersist; use checkpoint for lineage cutoff.
- Serialization: Kryo; tune `spark.sql.shuffle.partitions`.

BigQuery performance:
- Partitioning: By ingestion or timestamp; clustering on high‑cardinality columns (e.g., `customer_id`).
- Storage layout: Avoid nested SELECT on huge tables; use pruning via partition filters.
- Materialized views: Precompute heavy aggregations; scheduled queries for marts.
- Slot management: Reservations for critical jobs; monitor with Admin Insights.
- UDFs: Prefer SQL functions; avoid row‑by‑row JS UDFs.

Troubleshooting flow (ETL failure due to overload):
- Observe: Spark UI stages, shuffle read/write, GC, skew metrics.
- Identify: Skewed keys; too many partitions; tiny files; broadcast misfires.
- Fix: Rebalance (`repartition`), add salting, enable AQE, increase executor memory/cores wisely; compact outputs.
- Prevent: Compaction job; enforce partitioning standards; add data quality checks.

---

## Domain 3: Data Governance, Security, & Access

Principles:
- Least privilege (IAM roles) + conditions (ABAC) for context‑aware access.
- Sensitive data: Tokenize/hash or encrypt; separate raw from masked curated.
- Column/row level security: BigQuery policy tags for columns (e.g., PII), row access policies for subsets.
- Encryption: Default Google‑managed; use CMEK (Cloud KMS) for regulated workloads; VPC‑SC for perimeter.
- Auditing: Cloud Audit Logs; Dataplex data quality rules.

Techniques:
- Masking: Views that `CASE WHEN has_role('HR') THEN salary ELSE NULL END` or policy tags with column‑level access.
- Tokenization: Vault or deterministic hashing for joinability (e.g., `SHA256(salt || ssn)` with KMS‑protected salt).
- Secrets: Secret Manager; never store creds in code; use service accounts.
- Governance: Dataplex lakes/domains; Data Catalog tags; lineage with Cloud Data Lineage.

Scenario (HR visibility):
- Create BigQuery columns tagged with `PII.High` via policy tags; grant HR group `roles/bigquery.policyTagUser`.
- Others see aggregated metrics via views; enforce row access policies for department.
- Encrypt datasets with CMEK; enforce VPC‑SC; audit access.

---

## Domain 4: Practical SQL & Analytical Querying

Patterns to master:
- Window functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG/LEAD`, `SUM() OVER(...)`.
- CTEs: Break complex logic into readable steps.
- Time filters: Last month, week; handle timezone; use `DATE_TRUNC`.
- Arrays/Structs (BigQuery): `UNNEST`, `STRUCT` fields; array aggregations.
- Anti/semi joins: `LEFT JOIN ... WHERE right IS NULL`, `EXISTS`/`IN`.

Example: Top 3 highest‑spending customers per region last month (BigQuery)

```sql
WITH orders AS (
  SELECT *
  FROM `project.dataset.orders`
  WHERE order_ts >= DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 1 MONTH)
    AND order_ts < DATE_TRUNC(CURRENT_DATE(), MONTH)
)
, spend AS (
  SELECT customer_id, region, SUM(order_amount) AS total_spend
  FROM orders
  GROUP BY customer_id, region
)
SELECT region, customer_id, total_spend
FROM (
  SELECT region, customer_id, total_spend,
         ROW_NUMBER() OVER (PARTITION BY region ORDER BY total_spend DESC) AS rn
  FROM spend
)
WHERE rn <= 3
ORDER BY region, total_spend DESC;
```

---

# Mock Exam (90 minutes)

Instructions: Timebox each question; state assumptions; articulate trade‑offs.

## Section A — Architecture & Modeling (3 questions)

A1. Real‑Time Fraud Detection Architecture
- Design an end‑to‑end GCP architecture ingesting clickstream and transaction events (millions of users), scoring in real‑time, and providing analyst visibility.
- Requirements: Sub‑second detection for high‑risk events, replay capability, governance with sensitivity tags, DR across regions.
- Deliverables: Ingestion, processing, storage tiers, serving, governance, reliability.

Solution outline:
- Ingestion: Pub/Sub (events), Datastream (CDC).
- Processing: Dataflow with sliding windows, feature joins; dead‑letter topic.
- Storage: GCS raw Parquet; BigQuery curated features; Bigtable for hot features.
- Serving: Cloud Run scoring API; Looker Studio for monitoring.
- Governance: Dataplex domains; Data Catalog policy tags; CMEK; VPC‑SC.
- Reliability: Multi‑region topics; autoscaling; backlog replay from Pub/Sub; runbooks.
- Trade‑offs: Streaming cost vs latency; BigQuery vs Bigtable for features.

A2. Batch Lakehouse ETL with SCD2
- Design a nightly ETL that ingests dimension tables with SCD2 and fact tables into curated Parquet in GCS and marts in BigQuery.

Solution outline:
- Landing: Raw Avro/Parquet to GCS; schema registry in Dataplex.
- Transform: Dataproc Spark job computes SCD2 (`valid_from`, `valid_to`, `is_current`).
- Serve: BigQuery marts via `MERGE` into partitioned tables; materialized views.
- Ops: Compaction job; data quality checks; backfill strategy.

A3. Multi‑region DR for Data Lake
- Provide DR design for lakehouse ensuring RPO ≤ 15 min, RTO ≤ 1 hour.

Solution outline:
- GCS dual‑region buckets; Pub/Sub topic replication; Dataflow regional failover.
- BigQuery dataset replication; runbooks; IaC with Terraform; tests.

## Section B — Processing & Performance (4 questions)

B1. Spark Skew Troubleshooting
- A join between a 1TB fact and 10GB dimension is slow and fails with executor OOM. Diagnose and fix.

Solution:
- Diagnose via Spark UI: Shuffle read skew; single heavy partition.
- Fix: Broadcast dimension; salt skewed keys; enable AQE; adjust `spark.sql.shuffle.partitions`; pre‑aggregate.

B2. Small Files Problem
- You inherited millions of 5KB Parquet files in GCS. What’s your compaction plan and why?

Solution:
- Spark job to read partitions and write out 256MB files via `coalesce`; Dataflow or Dataproc; set `maxRecordsPerFile`; schedule compaction; update downstream partition expectations.

B3. BigQuery Slow Query
- Query scans 30TB with filter on `created_at` but still slow. How to optimize?

Solution:
- Ensure partitioning on `created_at`; cluster by high‑cardinality columns; use partition pruning; precompute with materialized views; verify slots/reservations; avoid `SELECT *`.

B4. ETL Failure Due to Overload
- Daily job failed after traffic spike. Outline systematic approach to prevent recurrence.

Solution:
- Observe metrics; find bottlenecks; scale reservations or Dataproc cluster; add backpressure in Dataflow; implement autoscaling and retries; compact outputs; add alerts.

## Section C — Governance & Security (3 questions)

C1. Column‑Level Security for Salary Data
- Ensure HR sees raw salary; others see masked or aggregated.

Solution:
- BigQuery policy tags on `salary` with HR group access; views expose masked columns for non‑HR; row access policies by department; audit logs; CMEK.

C2. PII Tokenization for Joinability
- You must join on email without revealing it.

Solution:
- Deterministic hash with salt (`SHA256(salt || LOWER(email))`); salt in Secret Manager/KMS; store hashed value for joins; rotate salt carefully; document re‑identification policy.

C3. Perimeter Security
- Prevent data exfiltration outside your project perimeter.

Solution:
- VPC‑SC around projects and services; restricted service accounts; egress controls; private endpoints; DLP scans.

## Section D — SQL (5 questions)

Assume BigQuery tables:
- `orders(order_id, customer_id, region, order_ts TIMESTAMP, order_amount NUMERIC)`
- `customers(customer_id, region, signup_ts TIMESTAMP)`
- `events(event_id, customer_id, event_ts TIMESTAMP, event_type STRING)`

D1. Top 3 highest‑spending customers per region last month.

Solution:
```sql
WITH last_month AS (
  SELECT * FROM `project.dataset.orders`
  WHERE order_ts >= DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 1 MONTH)
    AND order_ts < DATE_TRUNC(CURRENT_DATE(), MONTH)
), spend AS (
  SELECT customer_id, region, SUM(order_amount) AS total_spend
  FROM last_month
  GROUP BY customer_id, region
)
SELECT region, customer_id, total_spend
FROM (
  SELECT region, customer_id, total_spend,
         ROW_NUMBER() OVER (PARTITION BY region ORDER BY total_spend DESC) AS rn
  FROM spend
)
WHERE rn <= 3
ORDER BY region, total_spend DESC;
```

D2. Monthly retention: customers with an event both this month and last month.

Solution:
```sql
WITH this_month AS (
  SELECT DISTINCT customer_id
  FROM `project.dataset.events`
  WHERE event_ts >= DATE_TRUNC(CURRENT_DATE(), MONTH)
    AND event_ts < DATE_ADD(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 1 MONTH)
), last_month AS (
  SELECT DISTINCT customer_id
  FROM `project.dataset.events`
  WHERE event_ts >= DATE_SUB(DATE_TRUNC(CURRENT_DATE(), MONTH), INTERVAL 1 MONTH)
    AND event_ts < DATE_TRUNC(CURRENT_DATE(), MONTH)
)
SELECT COUNT(*) AS retained
FROM this_month t
JOIN last_month l USING (customer_id);
```

D3. Rolling 7‑day average order amount per customer.

Solution:
```sql
SELECT customer_id,
       DATE(order_ts) AS dt,
       AVG(order_amount) OVER (
         PARTITION BY customer_id
         ORDER BY DATE(order_ts)
         RANGE BETWEEN INTERVAL 6 DAY PRECEDING AND CURRENT ROW
       ) AS avg7
FROM `project.dataset.orders`;
```

D4. Identify first purchase date and days to second purchase per customer.

Solution:
```sql
WITH o AS (
  SELECT customer_id, order_ts,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_ts) AS rn
  FROM `project.dataset.orders`
)
SELECT customer_id,
       MIN(CASE WHEN rn = 1 THEN DATE(order_ts) END) AS first_purchase_date,
       DATE_DIFF(
         MIN(CASE WHEN rn = 2 THEN DATE(order_ts) END),
         MIN(CASE WHEN rn = 1 THEN DATE(order_ts) END),
         DAY
       ) AS days_to_second
FROM o
GROUP BY customer_id;
```

D5. Customers with no orders in the last 90 days but with events.

Solution:
```sql
WITH active_events AS (
  SELECT DISTINCT customer_id
  FROM `project.dataset.events`
  WHERE event_ts >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
), recent_orders AS (
  SELECT DISTINCT customer_id
  FROM `project.dataset.orders`
  WHERE order_ts >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 90 DAY)
)
SELECT customer_id
FROM active_events ae
LEFT JOIN recent_orders ro USING (customer_id)
WHERE ro.customer_id IS NULL;
```

---

## Behavioral & Communication (15–30 minutes)

- Narratives: Two impactful projects; your role; metrics; trade‑offs; failures and learnings.
- System design style: Clarify requirements; propose MVP and extensions; discuss risks; sketch data flows.
- Communication: Structure answers, state assumptions, use checklists, summarize.

## Final Rehearsal Checklist

- Architecture: Zones, formats, streaming + batch, catalog, DR.
- Performance: Partitioning, shuffles, skew fixes, file sizing, BigQuery tuning.
- Security: IAM/ABAC, policy tags, masking, CMEK, VPC‑SC, auditing.
- SQL: Windows, CTEs, last‑month filters, anti/semi joins, arrays.

Good luck! Focus on clarity, trade‑offs, and practical patterns.
