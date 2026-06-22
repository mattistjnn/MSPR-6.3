# Présentation de la Gestion de Projet
## NordTransit Logistics — Mission MSPR 2025-2026
**Version :** 1.0 | **Date :** Juin 2026

---

## 1. Organisation de l'équipe

### 1.1 Rôles et responsabilités

| Personne | Rôle | Responsabilités principales |
|----------|------|----------------------------|
| **Personne 1** | Architecture & Infrastructure | Choix SGBD, VMs Proxmox, réplication, PRA, RunBook, sauvegardes |
| **Personne 2** | Base de données & SQL | MCD/MLD, requêtes SQL, optimisation BDD, index, scripts |
| **Personne 3** | Sécurité & Supervision | Politiques d'accès, durcissement, monitoring, analyse logs |
| **Personne 4** | Gestion de projet & Direction | Planning, jalons, registre risques, note direction, soutenance |

### 1.2 Matrice RACI

| Livrable | P1 | P2 | P3 | P4 |
|----------|----|----|----|----|
| Choix SGBD | **R** | C | I | A |
| Architecture HA | **R** | I | C | A |
| MCD/MLD | C | **R** | I | A |
| Index et optimisations | C | **R** | I | A |
| Politique d'accès | C | C | **R** | A |
| Guide supervision | C | C | **R** | A |
| PRA | **R** | C | C | A |
| RunBook | **R** | C | C | A |
| Note de direction | C | I | C | **R** |
| Gestion projet | I | I | I | **R** |
| Soutenance | C | C | C | **R** |

*(R=Responsable, A=Approbateur, C=Consulté, I=Informé)*

---

## 2. Planification

### 2.1 Jalons du projet

| # | Jalon | Date cible | Statut |
|---|-------|-----------|--------|
| J1 | Kick-off équipe + lecture cahier des charges | Jour 1 | ✅ Terminé |
| J2 | Choix SGBD validé + architecture définie | Jour 2 | ✅ Terminé |
| J3 | Infrastructure Proxmox opérationnelle (vmbr5/6) | Jour 3 | ✅ Terminé |
| J4 | VMs créées et configurées (DB-01, DB-02, MGT) | Jour 4 | ✅ Terminé |
| J5 | Réplication Primary→Replica fonctionnelle | Jour 5 | ✅ Terminé |
| J6 | Scripts de sauvegarde testés et validés | Jour 6 | ✅ Terminé |
| J7 | Tests HA (failover manuel validé) | Jour 7 | 🔄 En cours |
| J8 | Documentation complète rédigée | Jour 8–9 | 🔄 En cours |
| J9 | Relecture + corrections finales | Jour 9 | ⬜ À faire |
| J10 | Support soutenance finalisé | Jour 10 | ⬜ À faire |

### 2.2 Planning de séquencement (Gantt simplifié)

```
Semaine 1
         J1  J2  J3  J4  J5  J6  J7
P1       ████████████████████████
P2       ████████████████
P3            ████████████████
P4       ████          ████████████

Semaine 2
         J8  J9  J10
P1       ████████████  Documentation + Soutenance
P2       ████████████
P3       ████████████
P4       ████████████
```

### 2.3 Charges estimées

| Tâche | P1 | P2 | P3 | P4 | Total |
|-------|----|----|----|----|-------|
| Analyse contexte / architecture | 4h | 2h | 2h | 2h | 10h |
| Infrastructure (VMs, réseau) | 6h | 1h | 1h | 0h | 8h |
| PostgreSQL (install, config, réplication) | 5h | 3h | 1h | 0h | 9h |
| MCD/MLD + SQL | 1h | 6h | 1h | 0h | 8h |
| Sécurité + supervision | 1h | 1h | 5h | 0h | 7h |
| Scripts sauvegarde | 2h | 2h | 1h | 0h | 5h |
| Tests + validation | 2h | 2h | 2h | 1h | 7h |
| Documentation | 4h | 3h | 3h | 4h | 14h |
| Soutenance | 1h | 1h | 1h | 3h | 6h |
| **Total** | **26h** | **21h** | **17h** | **10h** | **74h** |

---

## 3. Suivi d'avancement

### 3.1 Liste des tâches avec statut

| ID | Tâche | Responsable | Statut | Commentaire |
|----|-------|------------|--------|-------------|
| T01 | Lecture cahier des charges + contexte NTL | Tous | ✅ Terminé | |
| T02 | Comparatif MySQL vs PostgreSQL | P1 | ✅ Terminé | PostgreSQL 16 retenu |
| T03 | Création vmbr5 et vmbr6 sur Proxmox | P1 | ✅ Terminé | vmbr5: 192.168.60.0/24 |
| T04 | Création VM MSPR-DB-01 (Primary) | P1 | ✅ Terminé | pve-node1 |
| T05 | Création VM MSPR-DB-02 (Replica) | P1 | ✅ Terminé | pve-node2 |
| T06 | Création VM MSPR-MGT (Monitoring) | P1 | ✅ Terminé | pve-node1 |
| T07 | Configuration IP fixe + réseau VMs | P1 | ✅ Terminé | Netplan + NAT WireGuard |
| T08 | Installation PostgreSQL 16 DB-01 | P1 | ✅ Terminé | Depuis pgdg |
| T09 | Configuration réplication WAL (DB-01→DB-02) | P1 | ✅ Terminé | pg_basebackup + standby |
| T10 | Validation réplication (pg_stat_replication) | P1 | ✅ Terminé | streaming async OK |
| T11 | MCD NTL (entités WMS) | P2 | ✅ Terminé | 11 tables |
| T12 | MLD PostgreSQL (DDL SQL) | P2 | ✅ Terminé | Avec contraintes ACID |
| T13 | Index et vues matérialisées | P2 | ✅ Terminé | +208x perf stock |
| T14 | Politique d'accès (rôles PostgreSQL) | P3 | ✅ Terminé | Moindre privilège |
| T15 | Guide supervision (5 KPIs) | P3 | ✅ Terminé | |
| T16 | Script backup_full.sh | P1 | ✅ Terminé | Cron 00h00 |
| T17 | Script test_restore.sh | P1 | ✅ Terminé | Cron samedi 03h00 |
| T18 | PRA (scénarios + procédures) | P1 | ✅ Terminé | RTO 1h / RPO 15min |
| T19 | Test failover manuel DB-01→DB-02 | P1, P3 | 🔄 En cours | |
| T20 | Document architecture technique complet | P1, P2 | ✅ Terminé | |
| T21 | RunBook d'exploitation | P1 | ✅ Terminé | |
| T22 | Note de direction | P4 | ✅ Terminé | |
| T23 | Présentation soutenance | P4 | 🔄 En cours | |
| T24 | Relecture collective + corrections | Tous | ⬜ À faire | Jour 9 |

---

## 4. Registre des risques projet

| # | Risque | Probabilité | Impact | Mesure de mitigation | Responsable |
|---|--------|------------|--------|---------------------|-------------|
| R1 | **Incompatibilité versions PostgreSQL** (pg_basebackup entre versions différentes) | Faible | Haut | Vérifier versions avant toute migration ; utiliser pg_upgrade si nécessaire | P1 |
| R2 | **Réseau instable entre pve-node1 et pve-node2** (lag réplication excessif) | Moyen | Critique | Monitoring lag continu ; seuil alerte 50 Mo ; procédure resynchronisation documentée | P1, P3 |
| R3 | **Espace disque insuffisant** pour la croissance des données WMS + WAL | Moyen | Haut | Politique rétention 7/30/90j ; alertes > 70% ; extension disque planifiée | P1 |
| R4 | **Perte simultanée des deux nœuds Proxmox** | Très faible | Critique | Sauvegarde pg_basebackup hebdomadaire sur NAS ; test restauration mensuel | P1 |
| R5 | **Ransomware / chiffrement de la BDD** | Faible | Critique | Sauvegardes immuables (lecture seule sur NAS) ; isolation réseau vmbr5 ; accès SSH restreint | P3 |
| R6 | **Manque de temps pour documentation complète** | Moyen | Moyen | Répartition claire par personne ; templates pré-rédigés ; revue quotidienne avancement | P4 |
| R7 | **Indisponibilité d'un membre de l'équipe** | Faible | Moyen | Documentation des travaux en cours ; partage des accès dans le groupe ; backup opérationnel | P4 |

---

## 5. Journal des décisions

### Arbitrage 1 — PostgreSQL vs MySQL (Jour 1)

**Contexte :** NTL utilise MySQL en production. Deux options : conserver MySQL (migration minimale) ou migrer vers PostgreSQL (rupture technologique).

**Arguments pour MySQL :**
- Déjà présent dans l'infrastructure
- Moins de risque de migration
- Équipe potentiellement familiarisée

**Arguments pour PostgreSQL :**
- Réplication WAL native plus robuste que binlog
- Meilleure gestion des contraintes ACID pour un WMS
- Patroni (failover auto) mieux intégré avec PostgreSQL
- PITR natif (WAL archiving)
- Row Level Security pour isolation multi-clients

**Décision :** PostgreSQL 16 retenu à l'unanimité.
**Justification principale :** L'intégrité des données et la robustesse du PRA priment sur la continuité technologique. Le contexte NTL (WMS critique, fenêtres de maintenance courtes) exige la solution la plus fiable.

---

### Arbitrage 2 — Localisation des VMs (Jour 2)

**Contexte :** Deux options pour la répartition des VMs sur les deux nœuds Proxmox.

**Option A** — Tout sur pve-node1 (simplicité réseau)
- DB-01, DB-02, MGT sur pve-node1
- Risque : perte pve-node1 = perte totale

**Option B** — Répartition sur deux nœuds (résilience)
- DB-01 + MGT sur pve-node1 | DB-02 sur pve-node2
- Avantage : si pve-node1 tombe, DB-02 survive

**Décision :** Option B retenue — DB-02 sur pve-node2.
**Justification :** L'objectif RTO ≤ 1h n'est atteignable que si la Replica est sur un nœud physique différent. La complexité réseau supplémentaire (routage vmbr5 inter-nœuds) est acceptable.

---

### Arbitrage 3 — Synchrone vs Asynchrone pour la réplication (Jour 3)

**Contexte :** PostgreSQL propose deux modes : synchronous (RPO=0, impact perf) ou asynchronous (RPO=quelques secondes, perf optimale).

**Option A — Synchrone :**
- RPO = 0 (aucune perte de données)
- Impact : +10-30ms par transaction (Primary attend confirmation Replica)
- Risque : si Replica tombe, Primary se bloque (sauf synchronous_standby_names)

**Option B — Asynchrone :**
- RPO = lag réplication (typiquement < 1 seconde, objectif ≤ 15 min)
- Aucun impact perf sur Primary
- Replica peut décrocher en cas de réseau instable

**Décision :** Asynchrone retenu avec monitoring strict du lag.
**Justification :** Le cahier des charges exige RPO ≤ 15 min, pas RPO = 0. L'impact de la réplication synchrone sur les opérations WMS (300+ transactions/min en pointe) est inacceptable. Le monitoring du lag (seuil 50 Mo / alerte 500 Mo) garantit le respect du RPO.
