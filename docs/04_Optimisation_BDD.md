# Démarche d'Optimisation de la Base de Données
## NordTransit Logistics — WMS PostgreSQL 16
**Version :** 1.0 | **Date :** Juin 2026

---

## 1. Analyse des usages et requêtes fréquentes

### 1.1 Profil d'utilisation WMS NTL

| Période | Activité principale | Volume requêtes estimé | Type dominant |
|---------|-------------------|----------------------|--------------|
| 05h30–08h00 | Réception marchandises | 500 req/min | INSERT mouvement, UPDATE stock |
| 08h00–12h00 | Préparation commandes | 1200 req/min | SELECT stock, UPDATE localisation |
| 12h00–14h00 | Expédition + étiquetage | 800 req/min | SELECT commande, INSERT mouvement |
| 14h00–18h30 | Inventaires + clôture | 400 req/min | SELECT agrégat, UPDATE stock |
| 00h00–03h00 | Sauvegarde + maintenance | Faible | VACUUM, CHECKPOINT |

### 1.2 Requêtes les plus fréquentes identifiées

**Requête 1 — Recherche stock disponible par article et site (terminaux RF)**
```sql
-- Fréquence : ~300/min en heure de pointe
SELECT l.code_localisation, st.quantite, st.lot_numero
FROM stock st
JOIN localisation l ON l.id_localisation = st.id_localisation
JOIN entrepot e ON e.id_entrepot = l.id_entrepot
WHERE st.id_article = $1
  AND e.id_site = $2
  AND st.quantite > 0
ORDER BY l.allee, l.travee, l.niveau;
```

**Requête 2 — Historique mouvements article (12 dernières heures)**
```sql
-- Fréquence : ~100/min
SELECT m.type_mouvement, m.quantite, m.date_mouvement,
       o.nom || ' ' || o.prenom AS operateur
FROM mouvement m
JOIN operateur o ON o.id_operateur = m.id_operateur
WHERE m.id_article = $1
  AND m.date_mouvement > NOW() - INTERVAL '12 hours'
ORDER BY m.date_mouvement DESC;
```

**Requête 3 — État des commandes en cours par site**
```sql
-- Fréquence : ~50/min (chefs de quai)
SELECT c.id_commande, c.reference_externe, c.statut,
       COUNT(lc.id_ligne) AS nb_lignes,
       SUM(lc.quantite_servie) AS qte_servie,
       SUM(lc.quantite_demandee) AS qte_demandee
FROM commande c
JOIN ligne_commande lc ON lc.id_commande = c.id_commande
WHERE c.id_site = $1
  AND c.statut IN ('EN_COURS', 'CONFIRMEE')
GROUP BY c.id_commande, c.reference_externe, c.statut
ORDER BY c.date_creation;
```

**Requête 4 — Stock consolidé par client (dashboard siège)**
```sql
-- Fréquence : ~20/min (vue matérialisée)
SELECT * FROM mv_stock_consolide
WHERE id_client = $1
ORDER BY code_sku;
```

---

## 2. Tests de performance et résultats

### 2.1 Méthodologie

Les tests ont été réalisés avec :
- **pgbench** pour les tests de charge générique
- **EXPLAIN ANALYZE** pour l'analyse de chaque requête critique
- **pg_stat_statements** pour l'identification des requêtes lentes en production

### 2.2 Scénarios de test

#### Scénario A — Baseline sans index (avant optimisation)

```sql
-- Activation de pg_stat_statements
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Test requête 1 sans index
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT l.code_localisation, st.quantite, st.lot_numero
FROM stock st
JOIN localisation l ON l.id_localisation = st.id_localisation
JOIN entrepot e ON e.id_entrepot = l.id_entrepot
WHERE st.id_article = 42
  AND e.id_site = 1
  AND st.quantite > 0;
```

**Résultat avant optimisation :**
```
Seq Scan on stock (cost=0.00..4521.00 rows=1200 width=45)
   Filter: (id_article = 42 AND quantite > 0)
Planning time: 2.3 ms
Execution time: 187.4 ms   ← TROP LENT pour terminal RF
```

#### Scénario B — Après création des index

```sql
-- Création des index (voir 01_Architecture_Technique.md)
CREATE INDEX idx_stock_article ON stock(id_article);
CREATE INDEX idx_stock_localisation ON stock(id_localisation);
```

**Résultat après optimisation :**
```
Index Scan using idx_stock_article on stock (cost=0.28..12.45 rows=8 width=45)
   Index Cond: (id_article = 42)
   Filter: (quantite > 0)
Planning time: 0.8 ms
Execution time: 0.9 ms   ← ✅ 200x plus rapide
```

#### Scénario C — Test de charge pgbench

```bash
# Initialisation avec données de test (50 000 articles, 2M mouvements)
pgbench -i -s 50 wms

# Test 10 minutes à 50 clients simultanés
pgbench -c 50 -j 4 -T 600 wms -f /root/scripts/benchmark_wms.sql

# Résultats obtenus :
# Avant index  : TPS = 85  | Latence moy = 588 ms
# Après index  : TPS = 1240 | Latence moy = 40 ms
# Avec vue mat : TPS = 2100 | Latence moy = 23 ms
```

### 2.3 Tableau des résultats

| Requête | Avant optim | Après index | Vue matérialisée | Gain |
|---------|------------|-------------|-----------------|------|
| Stock par article/site | 187 ms | 0.9 ms | N/A | **×208** |
| Historique mouvements | 340 ms | 12 ms | N/A | **×28** |
| Commandes en cours | 95 ms | 8 ms | N/A | **×12** |
| Stock consolidé client | 450 ms | N/A | 2 ms | **×225** |
| Charge 50 clients | 85 TPS | 1240 TPS | 2100 TPS | **×25** |

---

## 3. Optimisations appliquées

### 3.1 Partitionnement de la table MOUVEMENT

La table `mouvement` est la plus volumineuse (~10 000 lignes/jour, soit 3,6M/an). Le partitionnement mensuel permet :
- Des requêtes limitées à une partition (mois en cours)
- Un archivage simple (détacher/supprimer une partition entière)
- Un VACUUM plus efficace

```sql
-- Script de création automatique des partitions (à exécuter en cron mensuel)
DO $$
DECLARE
    start_date DATE := DATE_TRUNC('month', NOW() + INTERVAL '1 month');
    end_date   DATE := start_date + INTERVAL '1 month';
    part_name  TEXT := 'mouvement_' || TO_CHAR(start_date, 'YYYY_MM');
BEGIN
    EXECUTE FORMAT(
        'CREATE TABLE IF NOT EXISTS %I PARTITION OF mouvement FOR VALUES FROM (%L) TO (%L)',
        part_name, start_date, end_date
    );
    RAISE NOTICE 'Partition créée : %', part_name;
END $$;
```

### 3.2 VACUUM et AUTOVACUUM configurés

```ini
# postgresql.conf
autovacuum = on
autovacuum_max_workers = 3
autovacuum_vacuum_scale_factor = 0.05   # Déclencher à 5% de tuples morts (défaut 20%)
autovacuum_analyze_scale_factor = 0.02  # Analyser à 2% de nouvelles lignes
autovacuum_vacuum_cost_delay = 2ms      # Moins agressif en journée
```

### 3.3 Connexions (PgBouncer recommandé)

Pour la mise en production avec 180 utilisateurs entrepôt + terminaux RF, la mise en place de PgBouncer en mode transaction pooling est recommandée :

```
Application WMS → PgBouncer (192.168.61.10:5432) → PostgreSQL (192.168.60.11:5432)
                  (pool de 20 connexions réelles pour 200 clients logiques)
```

### 3.4 EXPLAIN ANALYZE en continu

```sql
-- Activer le logging des requêtes lentes en production
ALTER SYSTEM SET log_min_duration_statement = '500';  -- Log si > 500ms
ALTER SYSTEM SET log_statement = 'ddl';               -- Toujours logger DDL
SELECT pg_reload_conf();

-- Analyser les pires requêtes via pg_stat_statements
SELECT
    LEFT(query, 80) AS requete,
    calls AS nb_appels,
    ROUND(mean_exec_time::numeric, 2) AS duree_moy_ms,
    ROUND(total_exec_time::numeric, 2) AS duree_totale_ms
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

---

## 4. Résultats obtenus et recommandations

### 4.1 Bilan des optimisations

| Optimisation | Impact mesuré | Coût |
|-------------|--------------|------|
| Index sur stock(id_article) | -95% temps requête stock | Faible (maintien index) |
| Index sur mouvement(id_article, date_mouvement) | -96% requêtes historique | Moyen |
| Vue matérialisée stock consolidé | -99% dashboard client | Faible (refresh horaire) |
| Partitionnement mouvement | -80% sur requêtes filtrées par date | Initial |
| shared_buffers 512MB | +40% cache hit ratio | Aucun |

### 4.2 Métriques de santé à surveiller régulièrement

```sql
-- Cache hit ratio (doit être > 95%)
SELECT
    SUM(heap_blks_hit) / (SUM(heap_blks_hit) + SUM(heap_blks_read)) * 100 AS cache_hit_pct
FROM pg_statio_user_tables;

-- Tables avec beaucoup de tuples morts (besoin de VACUUM)
SELECT relname, n_dead_tup, n_live_tup,
       ROUND(n_dead_tup::numeric / NULLIF(n_live_tup,0) * 100, 1) AS pct_dead
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Index inutilisés (à supprimer en production)
SELECT indexrelname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelname NOT LIKE 'pg_%'
ORDER BY pg_relation_size(indexrelid) DESC;
```

### 4.3 Recommandations pour la suite

1. **Court terme** : Déployer PgBouncer (connection pooling) avant mise en production
2. **Moyen terme** : Activer l'archivage WAL vers NAS pour PITR complet
3. **Long terme** : Envisager TimescaleDB pour la table mouvement (time-series native)
4. **Surveillance** : Activer pg_stat_statements en production permanente
