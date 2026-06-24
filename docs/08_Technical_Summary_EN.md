# Technical Summary — NordTransit Logistics WMS Database
## Conception, Exploitation and Protection of a Relational Database via a DBMS

**Project:** NTL Warehouse Management System — Database Migration & High Availability Setup
**Date:** June 2026 | **Classification:** Internal | **Version:** 1.0

---

## 1. Project Context

NordTransit Logistics (NTL) is a logistics SME based in northern France, operating a headquarters in Lille and three warehouses (Lens WH1, Valenciennes WH2, Arras WH3) plus a seasonal cross-dock site.

The WMS (Warehouse Management System) database is the operational core of the company. Any unavailability immediately halts reception and shipping operations across all four sites (05:30–18:30), impacting 180 warehouse operators and causing estimated losses of €5,000–€15,000 per hour.

The existing infrastructure relied on a single MySQL virtual machine with no replication, no automated backups, and no documented recovery procedures. This project addresses these critical risks.

---

## 2. Architecture Overview

### 2.1 Infrastructure

| VM | Role | Node | IP Address | Resources |
|----|------|------|-----------|-----------|
| MSPR-DB-01 | Primary PostgreSQL 16 | pve-node1 | 192.168.60.11 | 2 vCPU, 3 GB RAM, 30 GB |
| MSPR-DB-02 | Standby Replica (streaming) | pve-node2 | 192.168.60.12 | 2 vCPU, 2 GB RAM, 30 GB |
| MSPR-MGT | Monitoring + Backup | pve-node1 | 192.168.60.10 | 1 vCPU, 1 GB RAM, 10 GB |

Dedicated network: `vmbr5` — 192.168.60.0/24 (isolated, no internet exposure)

DB-01 and MGT are hosted on `pve-node1`; DB-02 is hosted on `pve-node2`. This physical separation ensures that a complete failure of one hypervisor node does not take down both the primary and standby databases.

### 2.2 High Availability

PostgreSQL 16 streaming replication (WAL-based) is active between DB-01 (primary) and DB-02 (standby). Replication is asynchronous, providing:

- **RPO ≤ 15 minutes** — maximum data loss in case of primary failure
- **RTO ≤ 1 hour** — maximum recovery time (manual failover tested and validated)

Manual failover was successfully tested on 24/06/2026:
1. DB-01 stopped → DB-02 promoted to primary (`pg_ctlcluster 16 main promote`)
2. Write operations confirmed on new primary
3. DB-01 rebuilt as new replica via `pg_basebackup` from DB-02
4. Streaming replication re-established

---

## 3. DBMS Selection: PostgreSQL 16 vs MySQL 8.0

| Criterion | PostgreSQL 16 | MySQL 8.0 | Decision |
|-----------|--------------|-----------|----------|
| Streaming replication | Native WAL | Binlog-based | PostgreSQL |
| ACID compliance | Full (DEFERRABLE constraints) | Partial (InnoDB) | PostgreSQL |
| Point-in-time recovery | Native PITR via WAL | Complex setup | PostgreSQL |
| Auto failover (Patroni) | Fully supported | Complex | PostgreSQL |
| Row-Level Security | Native | Not available | PostgreSQL |
| Partitioning | Range/List/Hash native | Partial | PostgreSQL |
| License cost | Free (open-source) | Free (community) | Equal |

**Decision: PostgreSQL 16** was selected for its superior data integrity, native WAL replication, and compatibility with high-availability tools (Patroni).

---

## 4. Data Model

### 4.1 Key Entities (WMS Schema)

The WMS schema covers 9 tables addressing all NTL business requirements:

| Table | Description | Key Constraints |
|-------|-------------|----------------|
| `client` | Customer data (data isolation per client) | UNIQUE siret |
| `site` | Warehouse sites (WH1, WH2, WH3, CDK, SIEGE) | CHECK type_site |
| `entrepot` | Physical warehouses per site | FK → site |
| `article` (SKU) | Product catalog with dimensions/weight | UNIQUE (id_client, code_sku) |
| `localisation` | Storage locations (aisle/bay/level) | UNIQUE code_localisation |
| `stock` | Current inventory by article and location | CHECK quantite >= 0 |
| `operateur` | Warehouse staff with roles | UNIQUE login |
| `commande` | Inbound/outbound orders | CHECK type_commande |
| `mouvement` | Timestamped stock movements (audit trail) | BIGSERIAL, FK → article/operateur |

### 4.2 Index Strategy

Critical indexes deployed for WMS performance:

```sql
CREATE INDEX idx_stock_article       ON stock(id_article);
CREATE INDEX idx_stock_localisation  ON stock(id_localisation);
CREATE INDEX idx_mvt_article_date    ON mouvement(id_article, date_mouvement DESC);
CREATE INDEX idx_mvt_operateur       ON mouvement(id_operateur);
CREATE INDEX idx_commande_statut     ON commande(statut, date_creation DESC);
```

Performance gain demonstrated via `EXPLAIN ANALYZE`: Bitmap Index Scan on `idx_stock_article`, execution time **0.037 ms** (vs. ~187 ms without index).

---

## 5. Security Policy — Least Privilege

| Role | Permissions | Used by |
|------|-------------|---------|
| `wms_app` | SELECT, INSERT, UPDATE, DELETE on wms schema | WMS application |
| `wms_readonly` | SELECT only on all tables | Reporting, dashboards |
| `repuser` | REPLICATION only | DB-02 replica |
| `backup_user` | pg_read_all_data, CREATEDB | Backup scripts on MGT |
| `postgres` | Superuser | DBA only — local access only |

All passwords use `scram-sha-256` authentication. Remote access is restricted to the `192.168.60.0/24` management network via `pg_hba.conf` rules.

---

## 6. Backup Strategy (3-2-1 Rule)

| Level | Method | Frequency | Retention | Destination |
|-------|--------|-----------|-----------|-------------|
| L1 — Full dump | `pg_dump` (pg_dump v16) | Daily 00:00 via cron | 7 days | `/backups/postgresql/` on MGT |
| L2 — WAL streaming | Continuous streaming replication | Real-time | ≤ 15 min RPO | DB-02 |
| L3 — Base backup | `pg_basebackup` | Weekly (Sunday 02:00) | 30 days | NAS |
| L4 — Offsite | Azure Blob Storage | Weekly | 90 days | Planned |

Backups are verified automatically each Saturday at 03:00 via `test_restore.sh`, which restores the latest dump to a test database on DB-01 and validates record counts.

---

## 7. Monitoring

A `health_check.sh` script deployed on MSPR-MGT monitors 6 indicators in real time:

| Indicator | Expected Value | Alert Level |
|-----------|---------------|-------------|
| Primary (DB-01) availability | Accepting connections | CRITICAL if down |
| Replica (DB-02) availability | Accepting connections | WARNING if down |
| Replication state | streaming | CRITICAL if not streaming |
| Replication lag | < 50 MB | WARNING > 50 MB, CRITICAL > 500 MB |
| Active connections | < 70 / 100 | WARNING > 70, CRITICAL > 90 |
| Last backup age | < 25 hours | WARNING > 25h, CRITICAL > 48h |

Output example (all green):
```
[OK]    Primary (DB-01 192.168.60.11) : EN LIGNE
[OK]    Replica (DB-02 192.168.60.12) : EN LIGNE
[OK]    Réplication : streaming actif
[OK]    Lag réplication : 0bytes
[OK]    Connexions actives : 7 / 100
[OK]    Dernière sauvegarde : wms_full_20260624_165859.sql.gz
```

---

## 8. Key Metrics Summary

| Metric | Target | Achieved |
|--------|--------|---------|
| RTO (Recovery Time Objective) | ≤ 1 hour | ~30 min (manual failover tested 24/06/2026) |
| RPO (Recovery Point Objective) | ≤ 15 min | WAL continuous streaming (lag: 0 bytes) |
| Daily backup | Automated | Daily at 00:00 via cron — confirmed OK |
| Replication lag | < 50 MB | 0 bytes (real-time) |
| Cache hit ratio | > 95% | 100% (measured via pg_statio_user_tables) |
| Index usage | Demonstrated | Bitmap Index Scan — 0.037 ms execution time |

---

*This document was produced as part of the MSPR TPRE623 — EPSI B3 ASRBD 2025-2026.*
*French documentation: see docs/01 through docs/07.*
