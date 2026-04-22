# Data Architecture

**Related:** [[solution-arch/concepts/cqrs]], [[solution-arch/concepts/event-sourcing]], [[solution-arch/concepts/acid-vs-base]], [[solution-arch/concepts/database-sharding]]

---

## Storage Type Selection

```
┌──────────────────────────────────────────────────────────┐
│                  Is data relational?                     │
└────────────┬─────────────────────────┬───────────────────┘
             │ Yes                     │ No
             ▼                         ▼
     Need ACID txns?          What's the access pattern?
        │        │                │          │          │
       Yes       No         Key lookups  Documents   Time series
        │        │                │          │          │
    PostgreSQL  MySQL        DynamoDB   MongoDB    InfluxDB
    CockroachDB (read        Redis       Couchbase  TimescaleDB
                replicas)
```

### Storage Types

| Type | Examples | Best for |
|------|---------|---------|
| Relational (OLTP) | PostgreSQL, MySQL | Transactional, structured, joins |
| Wide-column | Cassandra, HBase | Time-series, high write throughput |
| Document | MongoDB, Couchbase | Flexible schema, nested data |
| Key-value | Redis, DynamoDB | Fast lookup, cache, sessions |
| Graph | Neo4j, Amazon Neptune | Relationship-heavy queries |
| Search | Elasticsearch, OpenSearch | Full-text, faceted search |
| Time-series | InfluxDB, TimescaleDB | Metrics, IoT, monitoring |
| Object store | S3, GCS, Azure Blob | Files, backups, data lake |
| Columnar (OLAP) | BigQuery, Redshift, Snowflake | Analytics, aggregations |

---

## OLTP vs OLAP

```
OLTP (Online Transaction Processing)    OLAP (Online Analytical Processing)
────────────────────────────────────    ──────────────────────────────────────
Many short txns (ms)                    Few long queries (seconds–minutes)
Rows updated frequently                 Data mostly read-only
Optimised for writes                    Optimised for aggregations
Normalised schema (3NF)                 Denormalised (star/snowflake schema)
Current data                            Historical data
PostgreSQL, MySQL                       BigQuery, Redshift, Snowflake
```

Never run analytics queries directly on your OLTP database — it degrades transaction performance. Replicate to a data warehouse or use CQRS read models.

---

## Data Lake / Lakehouse / Warehouse

```
                     Raw data
                        │
          ┌─────────────▼─────────────┐
          │         Data Lake         │
          │  (object store: S3/GCS)   │
          │  all formats, all schemas │
          └─────────────┬─────────────┘
                        │ ETL / ELT
          ┌─────────────▼─────────────┐
          │      Data Warehouse       │
          │  (structured, governed)   │
          │  Redshift / BigQuery /    │
          │  Snowflake                │
          └─────────────┬─────────────┘
                        │
               BI Tools / Analysts
```

**Lakehouse:** Combines lake (raw, open format) with warehouse (ACID, schema enforcement). Delta Lake, Apache Iceberg, Apache Hudi.

---

## Data Mesh

Organisational approach: domain teams own their data as a product, exposing it via well-defined APIs or shared storage.

```
  Domain: Orders                Domain: Users
  ┌──────────────────┐          ┌──────────────────┐
  │  Orders Service  │          │  Users Service   │
  │  ┌────────────┐  │          │  ┌────────────┐  │
  │  │ Orders DB  │  │          │  │  Users DB  │  │
  │  └────────────┘  │          │  └────────────┘  │
  │  Data Product:   │          │  Data Product:   │
  │  orders_events   │          │  user_profiles   │
  └─────────┬────────┘          └─────────┬────────┘
            │                             │
            └──────────┬──────────────────┘
                       ▼
               Data Platform (infra)
           (self-serve: catalog, lineage, quality)
```

**vs Data Warehouse:** Centralised ownership vs federated ownership.

---

## Consistency Models

From strongest to weakest:

```
Linearisability (strongest)
│  Every read sees the most recent write. Feels like one machine.
│  Cost: high latency (need global coordination).
│
Sequential consistency
│  All nodes see operations in same order; may lag real time.
│
Causal consistency
│  Causally related operations appear in order; concurrent ops may vary.
│
Eventual consistency (weakest)
│  Given no new writes, all replicas will eventually converge.
│  Cost: may return stale data temporarily.
▼
```

**Rule of thumb:**
- Money / inventory → linearisable or strong consistency
- Social media feed / product reviews → eventual consistency is fine
- Shopping cart → causal consistency acceptable

## Sources
- [[solution-arch/sources/designing-data-intensive-applications]]
