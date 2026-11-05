# GridWatch Platform — Architecture Draft v0.3

**Status:** Draft for review

**Author:** Platform Engineering (previous engagement)

**Scope:** Centralised monitoring platform for EV charging networks. Multi-tenant across CPOs. Production target ~50,000 chargers / ~12,000 sites across India and Germany, telemetry every ~10 s.

> This document describes the proposed architecture for GridWatch. It is intended to be implementation-ready. Comments welcome.

---

## 1. Goals & Non-Functional Targets

GridWatch is sold on data freshness. The contractual KPIs are:

| Signal | Target | Percentile |
|---|---|---|
| Connector / charger status | < 5 s | P99.9 |
| Active session power & energy | < 10 s | P99.9 |
| Fault / alarm surfaced | < 5 s | P99.9 |
| Dashboard tile end-to-end | < 2 s | P99.9 |
| Map live-status update | < 10 s | P95 |

The platform must be highly available (we target 99.9% monthly) and must isolate each CPO's data from every other CPO's.

---

## 2. High-Level Architecture

```
Vendor CSMS  ──(poll API + webhook)──▶  Ingestion Service (Node/NestJS)
                                              │
                                              ▼
                                      Kafka topic: telemetry
                                              │
                          ┌───────────────────┼───────────────────┐
                          ▼                   ▼                   ▼
                   Status Consumer     Session Consumer     Settlement Consumer
                          │                   │                   │
                          └──────────────▶  PostgreSQL  ◀──────────┘
                                              ▲
                                              │
                                     Dashboard API (NestJS)
                                              ▲
                                              │
                                    Next.js Dashboard + Leaflet Map
```

We use a single PostgreSQL instance as the system of record for everything — chargers, sites, sessions, telemetry, invoices. Keeping one database keeps operations simple and avoids cross-store consistency problems.

---

## 3. Ingestion

The Ingestion Service registers a webhook with each vendor CSMS and additionally polls the `GET /v1/chargers` endpoint every 30 seconds as a backstop. Incoming events (from both the webhook and the poll) are published to a single Kafka topic, `telemetry`, partitioned by `charger_id`. Kafka gives us at-least-once delivery, which guarantees we never lose an event.

Consumers read from `telemetry` and write to PostgreSQL. Each consumer type handles one concern (status, sessions, settlement).

Because we poll every 30 seconds and also receive webhooks within 60 seconds, dashboard data is always fresh enough to meet the < 5 s and < 2 s KPIs above.

---

## 4. Storage

### 4.1 Operational tables

`operators`, `sites`, `chargers`, `connectors`, `sessions`, `invoices` — standard relational model in PostgreSQL with foreign keys between them.

`sites` stores `latitude` and `longitude` as `double precision` columns, plus the address string from the vendor.

### 4.2 Telemetry

All raw telemetry lands in one table:

```sql
CREATE TABLE telemetry (
  charger_id   text,
  ts           timestamptz,
  payload      jsonb
);
CREATE INDEX idx_telemetry_charger ON telemetry (charger_id);
```

We keep all telemetry indefinitely — operators occasionally want to look back over history, and storage is cheap. No partitioning is required; PostgreSQL handles large tables fine with the index above.

---

## 5. Dashboard & Map

### 5.1 Map

The map is the heart of the product. On load, the dashboard calls `GET /sites`, which returns the full GeoJSON `FeatureCollection` of every site the operator owns, with each site's current status embedded. The frontend renders each site as a Leaflet marker. As status changes arrive, the frontend polls `GET /sites` every 10 seconds and re-renders the markers.

"Nearest available charger to a point" is computed with:

```sql
SELECT charger_id
FROM chargers c JOIN sites s ON c.site_id = s.id
WHERE c.status = 'available'
ORDER BY (s.latitude - :lat)^2 + (s.longitude - :lng)^2
LIMIT 1;
```

Coverage / density (chargers per km² by region) is computed on demand from the same `sites` and `chargers` tables.

### 5.2 Dashboard tiles

Dashboard tiles (active sessions, faults, utilisation) query PostgreSQL directly on each page load and on each 10-second refresh, reading from the same operational tables the consumers are writing to.

---

## 6. Multi-Tenancy

All CPOs share the same tables. Every row carries an `operator_id`. The Dashboard API scopes queries by the `operator_id` that the frontend sends as a query parameter, e.g. `GET /sites?operator_id=acme`. This keeps the API simple and lets us add operators without schema changes.

---

## 7. Sessions & Settlement

A charging session emits a `session.start` and a `session.stop` event. The Session Consumer writes the session row. When a `session.stop` arrives, the Settlement Consumer reads it from the `telemetry` topic, computes energy delivered (`stop_meter_wh - start_meter_wh`), applies the CPO tariff, converts to local currency, and writes one row to `invoices`.

Because each session produces exactly one `session.stop` event, no deduplication is needed on the settlement side — one stop event, one invoice.

A daily settlement job at 02:00 IST sums the previous day's invoices per CPO and produces the reconciliation report.

---

## 8. Caching & Sessions

A Redis instance holds:
- Dashboard user session state (login sessions).
- A live-status cache keyed by `charger_id`, refreshed by the Status Consumer.

If Redis restarts, users re-authenticate and the cache repopulates from the next poll cycle. We run one Redis instance; Redis is fast and a single node comfortably handles our load.

---

## 9. High Availability & Disaster Recovery

PostgreSQL runs with **streaming replication in asynchronous mode** to a warm standby in a second availability zone.

- **RPO: 0** (no data loss).
- **RTO: 5 minutes.**

On primary failure, the on-call engineer is paged and promotes the standby manually. This keeps failover under human control and avoids the risk of an automated system promoting incorrectly.

---

## 10. Capacity Sizing

- 50,000 chargers × 1 reading / 10 s ≈ **5,000 writes/sec** into the `telemetry` table.
- Each telemetry row is roughly **2 KB** (the `jsonb` payload plus keys).
- That is 5,000 × 2 KB ≈ **10 MB/sec**, or about **864 GB/day**.

We will run PostgreSQL on a single **4 vCPU / 8 GB RAM** instance with a **1 TB SSD**. This handles the write rate comfortably and gives us roughly a **year** of telemetry headroom before we need to think about storage. We can scale the instance up if needed.

---

## 11. Security

- TLS on all external endpoints.
- Operators authenticate via Keycloak (OAuth2).
- Vendor webhooks are received on a public HTTPS endpoint and validated against a shared secret in the request header.

---

## 12. Observability

We ship Prometheus + Grafana with dashboards for:
- Host CPU, RAM, and disk usage.
- PostgreSQL connection count.
- HTTP request rate and 5xx rate on the Dashboard API.

This gives operations full visibility into platform health.

---

## 13. Open Questions

- Final choice of map basemap provider.
- Whether to add a second Kafka broker for redundancy in a later phase.
- Tariff model details per CPO (pending commercial input).
