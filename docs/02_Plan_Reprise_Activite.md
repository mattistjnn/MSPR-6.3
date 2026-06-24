# Plan de Reprise d'Activité (PRA)
## NordTransit Logistics — Base WMS PostgreSQL 16
**Version :** 1.0 | **Date :** Juin 2026 | **Classification :** Confidentiel

---

## 1. Objectifs et périmètre

### 1.1 Définitions

| Indicateur | Valeur cible | Justification |
|-----------|-------------|---------------|
| **RTO** (Recovery Time Objective) | ≤ 1 heure | Fenêtre opérationnelle 5h30–18h30 : chaque heure d'arrêt = blocage 4 sites |
| **RPO** (Recovery Point Objective) | ≤ 15 minutes | Archivage WAL continu ; perte max 15 min de transactions |

### 1.2 Périmètre couvert

- Base de données WMS PostgreSQL 16 (MSPR-DB-01, Primary)
- Réplication vers MSPR-DB-02 (Replica, pve-node2)
- VM MSPR-MGT (Monitoring, Patroni, etcd)
- Scripts de sauvegarde et archives WAL

### 1.3 Activités critiques impactées

| Activité | Impact si BDD indisponible | Délai critique |
|----------|--------------------------|----------------|
| Réception marchandises | Blocage total — aucun enregistrement | Immédiat |
| Préparation commandes | Blocage total — terminaux RF inertes | Immédiat |
| Expédition / étiquetage | Blocage impression — camions bloqués | < 15 min |
| Échanges EDI transporteurs | Interruption flux électroniques | < 30 min |
| Consultation stock siège | Dégradée (lecture seule possible) | Non bloquant |

---

## 2. Architecture de reprise

### 2.1 Scénarios de sinistre et solutions

| # | Scénario | Probabilité | Solution PRA | RTO estimé |
|---|----------|-------------|-------------|-----------|
| S1 | Crash PostgreSQL sur DB-01 | Moyen | Redémarrage automatique (systemd) | < 5 min |
| S2 | Perte de la VM MSPR-DB-01 | Faible | Promotion Replica DB-02 (Patroni) | < 30 min |
| S3 | Perte du nœud pve-node1 entier | Très faible | Promotion DB-02 + reconstruction MGT | < 1h |
| S4 | Corruption des données | Très faible | PITR depuis archives WAL | < 2h |
| S5 | Perte des deux nœuds | Exceptionnel | Restauration depuis NAS/Azure | < 4h |
| S6 | Ransomware / chiffrement BDD | Très faible | Restauration sauvegarde immutable | < 4h |

### 2.2 Flux de décision PRA

```
ALERTE DÉTECTÉE
      │
      ▼
PostgreSQL répond ?
   │         │
  OUI        NON
   │          │
   ▼          ▼
Vérifier    DB-01 accessible ?
 réplication   │         │
               OUI       NON
               │          │
               ▼          ▼
          Redémarrer   Replica OK ?
          PostgreSQL     │       │
          sur DB-01     OUI     NON
                         │       │
                         ▼       ▼
                      Promouvoir Restaurer
                      DB-02      depuis backup
                      en Primary
```

---

## 3. Procédures de reprise

### 3.1 Scénario S1 — Redémarrage PostgreSQL

**Durée estimée :** 2–5 minutes

```bash
# 1. Vérifier l'état
systemctl status postgresql@16-main

# 2. Consulter les logs
tail -50 /var/log/postgresql/postgresql-16-main.log

# 3. Redémarrer
systemctl restart postgresql@16-main

# 4. Vérifier la réplication (depuis DB-01)
sudo -u postgres psql -c "SELECT client_addr, state FROM pg_stat_replication;"

# 5. Confirmer aux équipes opérationnelles
```

### 3.2 Scénario S2 — Failover Replica (DB-01 perdu)

**Durée estimée :** 15–30 minutes

#### Phase 1 — Isolation (5 min)
```bash
# Sur MSPR-MGT : vérifier que DB-01 est vraiment mort
ping -c 5 192.168.60.11
psql -h 192.168.60.11 -U wms_readonly -c "SELECT 1;" 2>&1

# Confirmer que DB-02 est synchronisée
ssh admin@192.168.60.12
sudo -u postgres psql -c "SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();"
# Les deux doivent être égaux ou proches
```

#### Phase 2 — Promotion (5 min)
```bash
# Sur MSPR-DB-02 : promouvoir en Primary (commande Debian/Ubuntu)
sudo pg_ctlcluster 16 main promote
sleep 5

# Vérifier la promotion
sudo -u postgres psql -c "SELECT pg_is_in_recovery();"
# Doit retourner : f (false) = DB-02 est maintenant Primary
```

#### Phase 3 — Reconnexion applicative (5 min)
```bash
# Mettre à jour la configuration de l'application WMS
# Pointer les connexions vers 192.168.60.12:5432

# Tester la connexion
psql -h 192.168.60.12 -U wms_app -d wms -c "SELECT NOW();"
```

#### Phase 4 — Communication (5 min)
- Notifier le responsable d'entrepôt de chaque site
- Informer le chef de projet
- Ouvrir un incident dans le système de ticketing

#### Phase 5 — Reconstruction de DB-01 (dès que possible)
```bash
# Quand DB-01 est de nouveau disponible
ssh admin@192.168.60.11
systemctl stop postgresql@16-main
sudo -u postgres rm -rf /var/lib/postgresql/16/main/*

# Reconstruire depuis la nouvelle Primary (DB-02)
sudo -u postgres pg_basebackup \
  -h 192.168.60.12 \
  -U repuser \
  -D /var/lib/postgresql/16/main \
  -P -R --wal-method=stream
# Note : max_wal_senders doit être ≥ 10 dans postgresql.conf (DB-01) pour correspondre à DB-02

systemctl start postgresql@16-main

# Vérifier depuis DB-02 (nouveau Primary)
sudo -u postgres psql -c "SELECT client_addr, state FROM pg_stat_replication;"
```

### 3.3 Scénario S4 — Restauration PITR (corruption données)

**Durée estimée :** 1–2 heures

```bash
# 1. Identifier l'instant avant corruption
# Consulter les logs PostgreSQL
grep -i "error\|fatal\|corrupt" /var/log/postgresql/postgresql-16-main.log | tail -50

# 2. Arrêter PostgreSQL
systemctl stop postgresql@16-main

# 3. Sauvegarder l'état actuel (au cas où)
cp -a /var/lib/postgresql/16/main /var/lib/postgresql/16/main.corrupted

# 4. Restaurer le dernier pg_basebackup
sudo -u postgres rm -rf /var/lib/postgresql/16/main/*
sudo -u postgres tar -xzf /backups/basebackup/basebackup_YYYYMMDD.tar.gz \
  -C /var/lib/postgresql/16/main/

# 5. Configurer la restauration PITR
cat > /var/lib/postgresql/16/main/recovery.conf << EOF
restore_command = 'cp /backups/wal/%f %p'
recovery_target_time = '2026-06-22 14:30:00'  # avant la corruption
recovery_target_action = 'promote'
EOF

# 6. Démarrer en mode recovery
systemctl start postgresql@16-main

# 7. Suivre la progression
tail -f /var/log/postgresql/postgresql-16-main.log | grep -i "recovery\|restored"

# 8. Valider les données
sudo -u postgres psql -d wms -c "SELECT COUNT(*) FROM mouvement WHERE date_mouvement > NOW() - INTERVAL '1 hour';"
```

---

## 4. Tests de restauration

### 4.1 Plan de tests périodiques

| Test | Fréquence | Responsable | Durée max |
|------|-----------|-------------|-----------|
| Restauration pg_dump sur base test | Hebdomadaire (samedi 03h00) | Script automatique | 30 min |
| Failover Manuel DB-01 → DB-02 | Mensuel (dimanche 02h00) | DBA | 1 heure |
| PITR sur environnement de test | Trimestriel | DBA | 2 heures |
| PRA complet (simulation sinistre) | Semestriel | DBA + Responsable IT | Demi-journée |

### 4.2 Script de test automatique hebdomadaire

```bash
#!/bin/bash
# /root/backup_scripts/test_restore_weekly.sh

BACKUP_DIR="/backups/postgresql"
TEST_DB="wms_restore_test"
LOG="/var/log/backup_test.log"
LATEST=$(ls -t $BACKUP_DIR/wms_full_*.sql.gz | head -1)

log() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> $LOG; }

log "=== TEST RESTAURATION HEBDOMADAIRE ==="
log "Fichier: $LATEST"

# Créer base de test
sudo -u postgres dropdb --if-exists $TEST_DB
sudo -u postgres createdb $TEST_DB

# Restaurer
if zcat "$LATEST" | sudo -u postgres psql -d $TEST_DB -q; then
    # Vérifier l'intégrité
    NB_ARTICLES=$(sudo -u postgres psql -d $TEST_DB -t -c "SELECT COUNT(*) FROM article;")
    NB_MVTS=$(sudo -u postgres psql -d $TEST_DB -t -c "SELECT COUNT(*) FROM mouvement;")
    log "✅ Restauration OK — Articles: $NB_ARTICLES | Mouvements: $NB_MVTS"
    sudo -u postgres dropdb $TEST_DB
else
    log "❌ ERREUR restauration — ALERTE DBA REQUISE"
    # Envoyer alerte email
    echo "ALERTE: Échec test restauration $(date)" | mail -s "[NTL] ALERTE BACKUP" dba@ntl.fr
fi
```

---

## 5. Annuaire de crise PRA

| Rôle | Nom | Contact | Délai réponse |
|------|-----|---------|---------------|
| Responsable IT (N3) | [Nom] | [Tel] | < 30 min |
| DBA / Administrateur (N2) | [Nom] | [Tel] | < 15 min |
| Responsable entrepôt WH1 | [Nom] | [Tel] | Information |
| Responsable entrepôt WH2 | [Nom] | [Tel] | Information |
| Responsable entrepôt WH3 | [Nom] | [Tel] | Information |
| Astreinte technique | [Nom] | [Tel] | < 30 min (nuit) |

---

## 6. Indicateurs de succès PRA

Après toute opération de reprise, valider :

```bash
# 1. PostgreSQL répond
sudo -u postgres psql -c "SELECT version();"

# 2. Tables accessibles
sudo -u postgres psql -d wms -c "SELECT COUNT(*) FROM article; SELECT COUNT(*) FROM stock;"

# 3. Réplication active (si Replica présente)
sudo -u postgres psql -c "SELECT client_addr, state, sync_state FROM pg_stat_replication;"

# 4. Dernier mouvement récent
sudo -u postgres psql -d wms -c "SELECT MAX(date_mouvement) FROM mouvement;"

# 5. Connexion applicative
psql -h 192.168.60.11 -U wms_app -d wms -c "SELECT 1;" 2>&1 && echo "✅ App OK"
```
