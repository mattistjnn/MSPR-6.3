# MSPR 2025-2026 — NordTransit Logistics (NTL)
### Conception, exploitation et protection d'une base de données WMS

**Formation :** Administrateur Systèmes, Réseaux et Bases de Données  
**Blocs :** E6.3 — Gérer les données | E6.4 — Gérer un projet (approche DevOps/SysOps)  
**Contexte :** Migration et industrialisation de la base WMS de NordTransit Logistics vers PostgreSQL 16 en haute disponibilité

---

## Contexte du projet

NordTransit Logistics (NTL) est une PME logistique implantée dans les Hauts-de-France (siège à Lille, entrepôts à Lens, Valenciennes et Arras). La base de données WMS (Warehouse Management System) est critique : son indisponibilité provoque l'arrêt immédiat des opérations sur les 4 sites (5h30–18h30).

L'objectif de cette mission est de concevoir et industrialiser une nouvelle base WMS sur PostgreSQL 16 avec :
- **RTO ≤ 1 heure** (temps de remise en service)
- **RPO ≤ 15 minutes** (perte de données maximale)
- Architecture haute disponibilité sur cluster Proxmox
- Sauvegardes automatisées et testées
- Documentation opérationnelle complète

---

## Infrastructure déployée

| VM | Rôle | Nœud | IP |
|----|------|------|----|
| MSPR-DB-01 | Primary PostgreSQL 16 | pve-node1 | 192.168.60.11 |
| MSPR-DB-02 | Replica PostgreSQL 16 | pve-node2 | 192.168.60.12 |
| MSPR-MGT | Monitoring + Patroni + etcd | pve-node1 | 192.168.60.10 |

Réseau dédié : `vmbr5` — 192.168.60.0/24 (Management & Réplication)

---

## Livrables

| # | Document | Description |
|---|----------|-------------|
| 01 | [Architecture Technique](docs/01_Architecture_Technique.md) | MCD, MLD, justification PostgreSQL, index, hardware, politiques d'accès et de sauvegarde |
| 02 | [Plan de Reprise d'Activité](docs/02_Plan_Reprise_Activite.md) | 6 scénarios de sinistre, procédures de failover, PITR, tests de restauration |
| 03 | [Guide de Supervision](docs/03_Guide_Supervision.md) | 5 KPIs avec seuils d'alerte, procédures de remédiation, analyse de logs |
| 04 | [Optimisation BDD](docs/04_Optimisation_BDD.md) | Analyse des usages WMS, tests pgbench, résultats mesurés (×200 sur requêtes critiques) |
| 05 | [RunBook d'Exploitation](docs/05_RunBook_Exploitation.md) | Start/stop, checklists, procédure incident, matrice d'escalade N1/N2/N3, KPIs |
| 06 | [Gestion de Projet](docs/06_Gestion_Projet.md) | RACI, planning, suivi tâches, registre risques, journal des décisions |
| 07 | [Note de Direction](docs/07_Note_Direction.md) | Note non-technique comité de direction — risques cyber, impact métier, ROI |

---

## Équipe

| Personne | Rôle |
|----------|------|
| Personne 1 | Architecture & Infrastructure |
| Personne 2 | Base de données & SQL |
| Personne 3 | Sécurité & Supervision |
| Personne 4 | Gestion de projet & Direction |
