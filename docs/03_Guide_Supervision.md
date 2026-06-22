# Guide de Supervision
## NordTransit Logistics — Base WMS PostgreSQL 16
**Version :** 1.0 | **Date :** Juin 2026

---

## 1. Les 5 indicateurs critiques (KPIs)

### KPI 1 — Disponibilité PostgreSQL

| Paramètre | Valeur |
|-----------|--------|
| **Description** | État du service PostgreSQL (Primary et Replica) |
| **Seuil WARNING** | Replica déconnectée > 2 minutes |
| **Seuil CRITICAL** | Primary inaccessible |
| **Fréquence vérification** | Toutes les 30 secondes |
| **Commande** | `pg_isready -h 192.168.60.11 -p 5432` |

**Procédure de remédiation :**
```bash
# 1. Vérifier l'état du service
systemctl status postgresql@16-main

# 2. Consulter les logs
tail -100 /var/log/postgresql/postgresql-16-main.log | grep -i "error\|fatal"

# 3. Tenter le redémarrage
systemctl restart postgresql@16-main

# 4. Si échec → déclencher procédure PRA Scénario S1/S2
```

---

### KPI 2 — Lag de réplication

| Paramètre | Valeur |
|-----------|--------|
| **Description** | Retard de la Replica par rapport au Primary (en octets WAL) |
| **Seuil WARNING** | > 50 Mo de lag |
| **Seuil CRITICAL** | > 500 Mo de lag ou réplication interrompue |
| **Fréquence vérification** | Toutes les 60 secondes |

**Commande de vérification :**
```sql
-- Sur Primary (DB-01)
SELECT
    client_addr,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag_taille,
    EXTRACT(EPOCH FROM (now() - reply_time))::INT AS lag_secondes
FROM pg_stat_replication;
```

**Procédure de remédiation :**
```bash
# WARNING — Lag > 50 Mo
# 1. Vérifier la charge réseau entre pve-node1 et pve-node2
iperf3 -c 192.168.60.12 -t 10

# 2. Vérifier les requêtes longues sur Primary
sudo -u postgres psql -c "SELECT pid, now()-query_start AS durée, query FROM pg_stat_activity WHERE state = 'active' ORDER BY 2 DESC LIMIT 10;"

# 3. Vérifier les verrous bloquants
sudo -u postgres psql -c "SELECT * FROM pg_locks WHERE NOT granted;"

# CRITICAL — Réplication interrompue
# 1. Vérifier que DB-02 est joignable
ping -c 3 192.168.60.12

# 2. Sur DB-02 : vérifier le service
ssh admin@192.168.60.12 "systemctl status postgresql@16-main"

# 3. Consulter les logs DB-02
ssh admin@192.168.60.12 "tail -50 /var/log/postgresql/postgresql-16-main.log"

# 4. Si nécessaire, reconstruire la réplication
# → Procédure PRA Scénario S2
```

---

### KPI 3 — Espace disque

| Paramètre | Valeur |
|-----------|--------|
| **Description** | Utilisation disque sur /var/lib/postgresql et /backups |
| **Seuil WARNING** | > 70% |
| **Seuil CRITICAL** | > 85% |
| **Fréquence vérification** | Toutes les 5 minutes |

**Commandes de vérification :**
```bash
# Espace disque OS
df -h /var/lib/postgresql /backups

# Taille de la base
sudo -u postgres psql -c "SELECT pg_size_pretty(pg_database_size('wms')) AS taille_bdd;"

# Taille du WAL
du -sh /var/lib/postgresql/16/main/pg_wal/

# Top tables les plus volumineuses
sudo -u postgres psql -d wms -c "
SELECT relname, pg_size_pretty(pg_total_relation_size(oid)) AS taille
FROM pg_class WHERE relkind='r'
ORDER BY pg_total_relation_size(oid) DESC LIMIT 10;"
```

**Procédure de remédiation :**
```bash
# WARNING (70-85%)
# 1. Nettoyer les anciens WAL archivés (si archivage actif)
find /backups/wal -name "*.gz" -mtime +2 -delete

# 2. Nettoyer les sauvegardes pg_dump > 7 jours
find /backups/postgresql -name "wms_full_*.sql.gz" -mtime +7 -delete

# 3. VACUUM ANALYZE pour récupérer l'espace mort
sudo -u postgres psql -d wms -c "VACUUM ANALYZE;"

# CRITICAL (>85%)
# 1. Action immédiate : supprimer sauvegardes les plus anciennes
find /backups -name "*.gz" -mtime +3 -delete

# 2. Contacter responsable IT pour extension disque
# 3. Surveillance renforcée toutes les 10 min
```

---

### KPI 4 — Nombre de connexions actives

| Paramètre | Valeur |
|-----------|--------|
| **Description** | Connexions actives vs max_connections (100) |
| **Seuil WARNING** | > 70 connexions (70%) |
| **Seuil CRITICAL** | > 90 connexions (90%) |
| **Fréquence vérification** | Toutes les 30 secondes |

**Commandes de vérification :**
```sql
-- Connexions actives par état et application
SELECT
    state,
    application_name,
    count(*) AS nb,
    MAX(EXTRACT(EPOCH FROM (now() - query_start)))::INT AS max_duree_sec
FROM pg_stat_activity
WHERE pid <> pg_backend_pid()
GROUP BY state, application_name
ORDER BY nb DESC;

-- Connexions idle en attente (connexions zombies)
SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction'
AND query_start < NOW() - INTERVAL '5 minutes';
```

**Procédure de remédiation :**
```bash
# WARNING
# 1. Identifier les connexions longues et inutilisées
sudo -u postgres psql -c "SELECT pid, usename, application_name, state, now()-query_start AS duree FROM pg_stat_activity WHERE state IN ('idle','idle in transaction') ORDER BY 5 DESC LIMIT 20;"

# 2. Terminer les connexions zombies (idle in transaction > 10 min)
sudo -u postgres psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle in transaction' AND query_start < NOW() - INTERVAL '10 minutes';"

# CRITICAL
# 1. Couper les connexions idle immédiatement
sudo -u postgres psql -c "SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle' AND pid <> pg_backend_pid();"

# 2. Envisager d'ajouter PgBouncer (connection pooler)
```

---

### KPI 5 — Succès des sauvegardes

| Paramètre | Valeur |
|-----------|--------|
| **Description** | Résultat du dernier job de sauvegarde nocturne |
| **Seuil WARNING** | Sauvegarde manquante > 25h |
| **Seuil CRITICAL** | Sauvegarde échouée ou absente > 48h |
| **Fréquence vérification** | Toutes les heures |

**Commandes de vérification :**
```bash
# Dernière sauvegarde complète
ls -lt /backups/postgresql/wms_full_*.sql.gz | head -3

# Vérifier l'âge de la dernière sauvegarde
find /backups/postgresql -name "wms_full_*.sql.gz" -mtime -1 | wc -l
# Doit retourner >= 1

# Consulter le log de sauvegarde
tail -30 /var/log/backup_full.log

# Taille de la dernière sauvegarde (doit être > 0)
ls -lh $(ls -t /backups/postgresql/wms_full_*.sql.gz | head -1)
```

**Procédure de remédiation :**
```bash
# WARNING — Sauvegarde manquante
# 1. Vérifier le cron
crontab -l | grep backup

# 2. Lancer une sauvegarde manuelle immédiate
/root/backup_scripts/backup_full.sh

# 3. Vérifier l'espace disque
df -h /backups/

# CRITICAL — Échec répété
# 1. Vérifier les logs d'erreur
cat /var/log/backup_full.log | grep "ERREUR\|ERROR" | tail -20

# 2. Tester la connexion à PostgreSQL depuis le script
psql -h 192.168.60.11 -U backup_user -d wms -c "SELECT 1;"

# 3. Alerter le DBA immédiatement
echo "ALERTE SAUVEGARDE ÉCHOUÉE $(date)" | mail -s "[NTL][CRITIQUE] Backup Failed" dba@ntl.fr
```

---

## 2. Analyse de logs

### 2.1 Journaux pertinents

| Journal | Emplacement | Contenu |
|---------|-------------|---------|
| PostgreSQL | `/var/log/postgresql/postgresql-16-main.log` | Requêtes lentes, erreurs, connexions, réplication |
| Système (syslog) | `/var/log/syslog` | Événements OS, OOM killer, filesystem |
| Kernel | `journalctl -k` | Erreurs matérielles, réseau |
| Scripts backup | `/var/log/backup_full.log`, `/var/log/backup_test.log` | Résultats sauvegardes |
| Patroni | `journalctl -u patroni` | Changements de rôle, élections |
| WAL receiver | Dans postgresql-16-main.log | État streaming replication |

### 2.2 Patterns critiques à surveiller

```bash
# Erreurs critiques PostgreSQL
grep -E "FATAL|PANIC|ERROR" /var/log/postgresql/postgresql-16-main.log | tail -50

# Requêtes lentes (> 500ms configuré)
grep "duration:" /var/log/postgresql/postgresql-16-main.log | awk '{print $NF}' | sort -n | tail -20

# Problèmes de réplication
grep -i "replication\|wal\|streaming" /var/log/postgresql/postgresql-16-main.log | tail -30

# Tentatives de connexion échouées
grep "authentication failed\|password authentication failed" /var/log/postgresql/postgresql-16-main.log

# Checkpoints fréquents (signe de charge élevée)
grep "checkpoint" /var/log/postgresql/postgresql-16-main.log | tail -20

# Dépassements de connexions
grep "too many clients" /var/log/postgresql/postgresql-16-main.log

# OOM Killer (mémoire)
grep -i "oom\|out of memory\|killed process" /var/log/syslog | tail -20

# Erreurs disque
grep -i "I/O error\|read error\|write error" /var/log/syslog | tail -20
```

### 2.3 Script d'analyse quotidien automatisé

```bash
#!/bin/bash
# /root/scripts/daily_log_analysis.sh — exécuté à 06h00 chaque jour

LOG_PG="/var/log/postgresql/postgresql-16-main.log"
YESTERDAY=$(date -d "yesterday" '+%Y-%m-%d')
REPORT="/var/log/daily_report_$(date +%Y%m%d).txt"

{
echo "=== RAPPORT JOURNALIER NTL — $(date) ==="
echo ""
echo "--- Erreurs PostgreSQL (hier) ---"
grep "$YESTERDAY" $LOG_PG | grep -E "ERROR|FATAL|PANIC" | wc -l
echo " erreurs détectées"

echo ""
echo "--- Top 5 requêtes les plus lentes ---"
grep "$YESTERDAY" $LOG_PG | grep "duration:" | \
  sed 's/.*duration: //' | sort -rn | head -5

echo ""
echo "--- Connexions rejetées ---"
grep "$YESTERDAY" $LOG_PG | grep -c "authentication failed"

echo ""
echo "--- Checkpoints (nombre) ---"
grep "$YESTERDAY" $LOG_PG | grep -c "checkpoint complete"

echo ""
echo "--- État réplication ---"
sudo -u postgres psql -c "SELECT client_addr, state, pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) AS lag FROM pg_stat_replication;" 2>&1

echo ""
echo "--- Espace disque ---"
df -h /var/lib/postgresql /backups

} > $REPORT

# Envoyer par email
mail -s "[NTL] Rapport journalier DB $(date +%Y-%m-%d)" admin@ntl.fr < $REPORT
```

### 2.4 Corrélation d'événements

| Pattern détecté | Corrélation probable | Action |
|----------------|---------------------|--------|
| Checkpoints fréquents + requêtes lentes | Charge excessive, I/O saturé | Analyser pg_stat_bgwriter, réduire la charge |
| Auth failures + heure inhabituelle | Tentative d'intrusion | Vérifier fail2ban, alerter sécurité |
| Lag réplication croissant + réseau OK | Transaction longue bloquante | Identifier et tuer la transaction |
| OOM Killer + crash PostgreSQL | RAM insuffisante | Réduire shared_buffers, ajouter swap |
| Backup absent + disk full | Espace disque épuisé | Nettoyage d'urgence, extension disque |

---

## 3. Tableau de bord synthétique

```
╔══════════════════════════════════════════════════════════╗
║         SUPERVISION NTL-WMS — ÉTAT TEMPS RÉEL           ║
╠══════════════════════════════════════════════════════════╣
║  🟢 PostgreSQL Primary (DB-01)    : ONLINE               ║
║  🟢 PostgreSQL Replica (DB-02)    : STREAMING (lag: 0B)  ║
║  🟢 Connexions actives             : 23/100              ║
║  🟢 Espace disque /var/lib/pg      : 42% (12Go/30Go)     ║
║  🟢 Dernière sauvegarde            : 2026-06-22 00:03    ║
║  🟢 Dernier test restauration      : 2026-06-21 03:00 ✅ ║
╚══════════════════════════════════════════════════════════╝
```

**Script de contrôle rapide :**
```bash
#!/bin/bash
# /root/scripts/health_check.sh

echo "=== HEALTH CHECK NTL-WMS $(date) ==="

# PostgreSQL Primary
pg_isready -h 192.168.60.11 -p 5432 -q && echo "✅ Primary: OK" || echo "❌ Primary: DOWN"

# PostgreSQL Replica
pg_isready -h 192.168.60.12 -p 5432 -q && echo "✅ Replica: OK" || echo "⚠️ Replica: DOWN"

# Lag réplication
sudo -u postgres psql -h 192.168.60.11 -c \
  "SELECT pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)) FROM pg_stat_replication;" 2>/dev/null

# Connexions
sudo -u postgres psql -h 192.168.60.11 -c \
  "SELECT count(*) AS connexions_actives FROM pg_stat_activity;" 2>/dev/null

# Espace disque
df -h /var/lib/postgresql | tail -1 | awk '{print "💾 Disque DB: "$5" utilisé"}'

# Dernière sauvegarde
ls -lt /backups/postgresql/wms_full_*.sql.gz 2>/dev/null | head -1 | awk '{print "📦 Dernière backup: "$6,$7,$8}'
```
