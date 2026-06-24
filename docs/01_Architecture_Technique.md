# Document d'Architecture Technique
## NordTransit Logistics (NTL) — Système WMS PostgreSQL 16
**Version :** 1.0 | **Date :** Juin 2026 | **Auteur :** Mattis TAJAN — Architecture & Infrastructure

---

## 1. Justification du SGBD retenu

### 1.1 Contexte de migration

NTL exploite actuellement une base MySQL hébergée sur une VM Ubuntu (WMS-DB, 192.168.10.21). L'application WMS est critique : son indisponibilité provoque l'arrêt immédiat des opérations de réception et d'expédition sur les quatre sites (5h30–18h30). L'absence de réplication documentée, de tests de restauration et d'objectifs RTO/RPO formalisés constitue un risque opérationnel majeur.

### 1.2 Comparatif MySQL 8.0 vs PostgreSQL 16

| Critère | MySQL 8.0 | PostgreSQL 16 | Recommandation |
|---------|-----------|---------------|----------------|
| Contraintes référentielles | Partial (InnoDB) | ACID complet + DEFERRABLE | ✅ PostgreSQL |
| Réplication native HA | Binlog binaire | Streaming WAL natif | ✅ PostgreSQL |
| Support JSON | Basique | Avancé (jsonb, opérateurs, index GIN) | ✅ PostgreSQL |
| Full-text search | Limité | Avancé (tsvector/tsquery) | ✅ PostgreSQL |
| CTE récursifs | Limité | Complet (WITH RECURSIVE) | ✅ PostgreSQL |
| Partitionnement natif | Partiel | Range/List/Hash natif | ✅ PostgreSQL |
| Point-in-time recovery (PITR) | Complexe | Natif via WAL archiving | ✅ PostgreSQL |
| Failover automatique | Non natif | Patroni compatible | ✅ PostgreSQL |
| Migration depuis MySQL | N/A | pgLoader disponible | ✅ PostgreSQL |
| TCO (coût total) | Bas | Bas (licence libre) | = |
| Présence dans l'existant | ✅ Oui | Non | ✅ MySQL |
| Maturité production logistique | Bonne | Très bonne (Carrefour, SNCF, DHL) | ✅ PostgreSQL |

### 1.3 Décision : PostgreSQL 16

**PostgreSQL 16 est retenu** pour les raisons suivantes :

- **Intégrité des données** : contraintes ACID strictes essentielles pour les mouvements de stock (aucune perte de cohérence tolérée en logistique)
- **Réplication WAL** : streaming replication natif permettant d'atteindre RPO ≤ 15 min
- **PITR** : restauration à un instant précis en cas de corruption
- **Patroni** : failover automatique, RTO ≤ 1h atteignable
- **Évolutivité** : partitionnement natif pour la table MOUVEMENT (volumétrie croissante)
- **Conformité** : meilleure gestion de la séparation des données par client (Row Level Security)

**Risque MySQL identifié et rejeté :** réplication binlog moins fiable que WAL, recovery complexe en urgence, contraintes FK délicates.

---

## 2. Modèle Conceptuel de Données (MCD)

### 2.1 Entités et attributs

```
CLIENT
├── id_client (PK)
├── raison_sociale
├── siret
├── adresse, ville, code_postal
├── contact_nom, contact_email, contact_tel
└── date_creation

SITE
├── id_site (PK)
├── code_site (WH1/WH2/WH3/CDK/SIEGE)
├── nom_site
├── adresse, ville
├── type_site (entrepot/siege/crossdock)
└── statut (actif/inactif)

ENTREPOT
├── id_entrepot (PK)
├── id_site (FK → SITE)
├── code_entrepot
├── superficie_m2
├── nb_allees
└── capacite_totale_palettes

ARTICLE (SKU)
├── id_article (PK)
├── id_client (FK → CLIENT)
├── code_sku
├── designation
├── poids_kg
├── longueur_cm, largeur_cm, hauteur_cm
├── volume_m3 (calculé)
├── categorie
├── unite_mesure
├── stock_mini, stock_maxi
└── statut (actif/inactif/obsolete)

LOCALISATION
├── id_localisation (PK)
├── id_entrepot (FK → ENTREPOT)
├── code_localisation
├── allee, travee, niveau
├── type_localisation (rack/sol/vrac/frigo)
├── capacite_poids_kg
├── capacite_volume_m3
└── statut (libre/occupe/bloque/maintenance)

STOCK
├── id_stock (PK)
├── id_article (FK → ARTICLE)
├── id_localisation (FK → LOCALISATION)
├── quantite
├── lot_numero
├── date_peremption
├── date_entree
└── date_maj

MOUVEMENT
├── id_mouvement (PK)
├── type_mouvement (ENTREE/SORTIE/TRANSFERT/AJUSTEMENT)
├── id_article (FK → ARTICLE)
├── id_localisation_source (FK → LOCALISATION, nullable)
├── id_localisation_dest (FK → LOCALISATION, nullable)
├── quantite
├── lot_numero
├── id_operateur (FK → OPERATEUR)
├── date_mouvement (horodatage)
├── id_commande_ref (FK → COMMANDE, nullable)
└── commentaire

COMMANDE
├── id_commande (PK)
├── id_client (FK → CLIENT)
├── id_site (FK → SITE)
├── type_commande (ENTREE/SORTIE)
├── date_creation
├── date_prevue
├── date_realisee
├── statut (BROUILLON/CONFIRMEE/EN_COURS/TERMINEE/ANNULEE)
├── reference_externe
└── transporteur

LIGNE_COMMANDE
├── id_ligne (PK)
├── id_commande (FK → COMMANDE)
├── id_article (FK → ARTICLE)
├── quantite_demandee
├── quantite_servie
└── statut_ligne

OPERATEUR
├── id_operateur (PK)
├── id_site (FK → SITE)
├── login (unique)
├── nom, prenom
├── email
├── role (ADMIN/CHEF_QUAI/PREPARATEUR/RECEPTIONNAIRE/SUPERVISEUR)
├── date_creation
└── actif (boolean)

FOURNISSEUR
├── id_fournisseur (PK)
├── raison_sociale
├── siret
├── adresse, ville
└── contact_nom, contact_email, contact_tel
```

### 2.2 Relations principales

```
CLIENT ─── 1:N ──► ARTICLE         (un client possède plusieurs SKUs)
CLIENT ─── 1:N ──► COMMANDE        (un client passe plusieurs commandes)
SITE ────── 1:N ──► ENTREPOT       (un site a un ou plusieurs entrepôts)
SITE ────── 1:N ──► OPERATEUR      (opérateurs affectés à un site)
ENTREPOT ── 1:N ──► LOCALISATION   (un entrepôt a plusieurs emplacements)
ARTICLE ─── 1:N ──► STOCK          (un article stocké en plusieurs emplacements)
LOCALISATION ─ 1:N ► STOCK         (une localisation accueille plusieurs articles)
ARTICLE ─── 1:N ──► MOUVEMENT      (historique mouvements par article)
COMMANDE ── 1:N ──► LIGNE_COMMANDE
COMMANDE ── 1:N ──► MOUVEMENT      (mouvements liés à une commande)
OPERATEUR ─ 1:N ──► MOUVEMENT      (traçabilité opérateur)
```

---

## 3. Modèle Logique de Données (MLD)

```sql
-- Clients
CREATE TABLE client (
    id_client     SERIAL PRIMARY KEY,
    raison_sociale VARCHAR(200) NOT NULL,
    siret          CHAR(14) UNIQUE,
    adresse        VARCHAR(255),
    ville          VARCHAR(100),
    code_postal    CHAR(5),
    contact_nom    VARCHAR(100),
    contact_email  VARCHAR(150),
    contact_tel    VARCHAR(20),
    date_creation  DATE NOT NULL DEFAULT CURRENT_DATE,
    CONSTRAINT chk_siret CHECK (siret ~ '^\d{14}$')
);

-- Sites / Entrepôts
CREATE TABLE site (
    id_site    SERIAL PRIMARY KEY,
    code_site  VARCHAR(10) UNIQUE NOT NULL,  -- WH1, WH2, WH3, CDK, SIEGE
    nom_site   VARCHAR(150) NOT NULL,
    adresse    VARCHAR(255),
    ville      VARCHAR(100),
    type_site  VARCHAR(20) NOT NULL CHECK (type_site IN ('entrepot','siege','crossdock')),
    statut     VARCHAR(10) NOT NULL DEFAULT 'actif' CHECK (statut IN ('actif','inactif'))
);

CREATE TABLE entrepot (
    id_entrepot          SERIAL PRIMARY KEY,
    id_site              INT NOT NULL REFERENCES site(id_site),
    code_entrepot        VARCHAR(20) UNIQUE NOT NULL,
    superficie_m2        NUMERIC(8,2),
    nb_allees            INT,
    capacite_totale_palettes INT,
    CONSTRAINT chk_superficie CHECK (superficie_m2 > 0)
);

-- Articles / SKU
CREATE TABLE article (
    id_article    SERIAL PRIMARY KEY,
    id_client     INT NOT NULL REFERENCES client(id_client),
    code_sku      VARCHAR(50) NOT NULL,
    designation   VARCHAR(255) NOT NULL,
    poids_kg      NUMERIC(10,3) NOT NULL CHECK (poids_kg >= 0),
    longueur_cm   NUMERIC(8,2) CHECK (longueur_cm > 0),
    largeur_cm    NUMERIC(8,2) CHECK (largeur_cm > 0),
    hauteur_cm    NUMERIC(8,2) CHECK (hauteur_cm > 0),
    volume_m3     NUMERIC(10,6) GENERATED ALWAYS AS
                  (longueur_cm * largeur_cm * hauteur_cm / 1000000) STORED,
    categorie     VARCHAR(100),
    unite_mesure  VARCHAR(20) NOT NULL DEFAULT 'PIECE',
    stock_mini    INT DEFAULT 0 CHECK (stock_mini >= 0),
    stock_maxi    INT CHECK (stock_maxi >= stock_mini),
    statut        VARCHAR(15) NOT NULL DEFAULT 'actif'
                  CHECK (statut IN ('actif','inactif','obsolete')),
    UNIQUE (id_client, code_sku)
);

-- Localisations
CREATE TABLE localisation (
    id_localisation    SERIAL PRIMARY KEY,
    id_entrepot        INT NOT NULL REFERENCES entrepot(id_entrepot),
    code_localisation  VARCHAR(30) UNIQUE NOT NULL,
    allee              VARCHAR(10),
    travee             VARCHAR(10),
    niveau             VARCHAR(10),
    type_localisation  VARCHAR(20) NOT NULL
                       CHECK (type_localisation IN ('rack','sol','vrac','frigo','palette')),
    capacite_poids_kg  NUMERIC(8,2),
    capacite_volume_m3 NUMERIC(8,4),
    statut             VARCHAR(15) NOT NULL DEFAULT 'libre'
                       CHECK (statut IN ('libre','occupe','bloque','maintenance'))
);

-- Stock actuel
CREATE TABLE stock (
    id_stock        SERIAL PRIMARY KEY,
    id_article      INT NOT NULL REFERENCES article(id_article),
    id_localisation INT NOT NULL REFERENCES localisation(id_localisation),
    quantite        INT NOT NULL DEFAULT 0 CHECK (quantite >= 0),
    lot_numero      VARCHAR(50),
    date_peremption DATE,
    date_entree     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    date_maj        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (id_article, id_localisation, lot_numero)
);

-- Opérateurs
CREATE TABLE operateur (
    id_operateur  SERIAL PRIMARY KEY,
    id_site       INT NOT NULL REFERENCES site(id_site),
    login         VARCHAR(50) UNIQUE NOT NULL,
    nom           VARCHAR(100) NOT NULL,
    prenom        VARCHAR(100) NOT NULL,
    email         VARCHAR(150) UNIQUE NOT NULL,
    role          VARCHAR(25) NOT NULL
                  CHECK (role IN ('ADMIN','CHEF_QUAI','PREPARATEUR',
                                  'RECEPTIONNAIRE','SUPERVISEUR')),
    date_creation DATE NOT NULL DEFAULT CURRENT_DATE,
    actif         BOOLEAN NOT NULL DEFAULT TRUE
);

-- Commandes
CREATE TABLE commande (
    id_commande       SERIAL PRIMARY KEY,
    id_client         INT NOT NULL REFERENCES client(id_client),
    id_site           INT NOT NULL REFERENCES site(id_site),
    type_commande     VARCHAR(10) NOT NULL CHECK (type_commande IN ('ENTREE','SORTIE')),
    date_creation     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    date_prevue       DATE,
    date_realisee     TIMESTAMPTZ,
    statut            VARCHAR(15) NOT NULL DEFAULT 'BROUILLON'
                      CHECK (statut IN ('BROUILLON','CONFIRMEE','EN_COURS','TERMINEE','ANNULEE')),
    reference_externe VARCHAR(100),
    transporteur      VARCHAR(150)
);

CREATE TABLE ligne_commande (
    id_ligne           SERIAL PRIMARY KEY,
    id_commande        INT NOT NULL REFERENCES commande(id_commande) ON DELETE CASCADE,
    id_article         INT NOT NULL REFERENCES article(id_article),
    quantite_demandee  INT NOT NULL CHECK (quantite_demandee > 0),
    quantite_servie    INT DEFAULT 0 CHECK (quantite_servie >= 0),
    statut_ligne       VARCHAR(15) NOT NULL DEFAULT 'EN_ATTENTE'
                       CHECK (statut_ligne IN ('EN_ATTENTE','PARTIEL','COMPLET','ANNULE'))
);

-- Mouvements (table principale — partitionnée)
CREATE TABLE mouvement (
    id_mouvement           BIGSERIAL,
    type_mouvement         VARCHAR(15) NOT NULL
                           CHECK (type_mouvement IN ('ENTREE','SORTIE','TRANSFERT','AJUSTEMENT')),
    id_article             INT NOT NULL REFERENCES article(id_article),
    id_localisation_source INT REFERENCES localisation(id_localisation),
    id_localisation_dest   INT REFERENCES localisation(id_localisation),
    quantite               INT NOT NULL CHECK (quantite > 0),
    lot_numero             VARCHAR(50),
    id_operateur           INT NOT NULL REFERENCES operateur(id_operateur),
    date_mouvement         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    id_commande_ref        INT REFERENCES commande(id_commande),
    commentaire            TEXT,
    PRIMARY KEY (id_mouvement, date_mouvement)
) PARTITION BY RANGE (date_mouvement);

-- Partitions mensuelles pour 2026
CREATE TABLE mouvement_2026_01 PARTITION OF mouvement
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
CREATE TABLE mouvement_2026_06 PARTITION OF mouvement
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');
-- (à créer mensuellement via script cron)
```

---

## 4. Index et optimisations

### 4.1 Index définis

```sql
-- Stock : recherche fréquente par article et localisation
CREATE INDEX idx_stock_article    ON stock(id_article);
CREATE INDEX idx_stock_localisation ON stock(id_localisation);
CREATE INDEX idx_stock_lot        ON stock(lot_numero) WHERE lot_numero IS NOT NULL;

-- Mouvement : requêtes temporelles et par article (les plus fréquentes)
CREATE INDEX idx_mvt_article_date  ON mouvement(id_article, date_mouvement DESC);
CREATE INDEX idx_mvt_operateur     ON mouvement(id_operateur, date_mouvement DESC);
CREATE INDEX idx_mvt_commande      ON mouvement(id_commande_ref) WHERE id_commande_ref IS NOT NULL;
CREATE INDEX idx_mvt_type_date     ON mouvement(type_mouvement, date_mouvement DESC);

-- Article : recherche par SKU et client
CREATE INDEX idx_article_sku       ON article(code_sku);
CREATE INDEX idx_article_client    ON article(id_client);
CREATE INDEX idx_article_statut    ON article(statut) WHERE statut = 'actif';

-- Commande : filtrage par statut et date
CREATE INDEX idx_commande_statut   ON commande(statut, date_creation DESC);
CREATE INDEX idx_commande_client   ON commande(id_client, date_creation DESC);
CREATE INDEX idx_commande_site     ON commande(id_site, statut);

-- Localisation : recherche par code et entrepôt
CREATE INDEX idx_loc_entrepot      ON localisation(id_entrepot, statut);

-- Full-text search sur désignation article
CREATE INDEX idx_article_fts       ON article USING GIN(to_tsvector('french', designation));
```

### 4.2 Vues matérialisées

```sql
-- Vue : stock consolidé par article et site (rafraîchie toutes les heures)
CREATE MATERIALIZED VIEW mv_stock_consolide AS
SELECT
    a.id_client,
    a.code_sku,
    a.designation,
    s.id_site,
    si.code_site,
    SUM(st.quantite) AS quantite_totale,
    COUNT(DISTINCT st.id_localisation) AS nb_emplacements
FROM stock st
JOIN article a ON a.id_article = st.id_article
JOIN localisation l ON l.id_localisation = st.id_localisation
JOIN entrepot e ON e.id_entrepot = l.id_entrepot
JOIN site si ON si.id_site = e.id_site
GROUP BY a.id_client, a.code_sku, a.designation, s.id_site, si.code_site;

CREATE UNIQUE INDEX idx_mv_stock ON mv_stock_consolide(id_client, code_sku, id_site);

-- Rafraîchissement : cron toutes les heures (hors fenêtre critique)
-- 0 * * * * psql -U wms_app -d wms -c "REFRESH MATERIALIZED VIEW CONCURRENTLY mv_stock_consolide;"
```

### 4.3 Paramètres PostgreSQL optimisés

```ini
# postgresql.conf — paramètres production NTL
shared_buffers          = 512MB       # 25% de 2GB RAM (VM DB-01)
effective_cache_size    = 1536MB      # estimation cache OS
work_mem               = 8MB          # par connexion
maintenance_work_mem   = 128MB        # VACUUM/index rebuild
wal_level              = replica
max_wal_senders        = 10
max_replication_slots  = 10
wal_buffers            = 16MB
checkpoint_timeout     = 15min
checkpoint_completion_target = 0.9
log_statement          = 'ddl'        # prod: DDL seulement
log_min_duration_statement = 500      # requêtes > 500ms logguées
autovacuum             = on
```

---

## 5. Architecture d'hébergement et hardware

### 5.1 Infrastructure Proxmox

| Composant | Spécification | Localisation |
|-----------|--------------|--------------|
| Hyperviseur pve-node1 | Dell PowerEdge R630, 2×Xeon E5-2630v3, 128 Go RAM | Lille (Siège) |
| Hyperviseur pve-node2 | Serveur secondaire (même gamme) | Lille (Siège) |
| Stockage | NAS 6 To RAID5, SAS 10K | Lille (Siège) |
| Réseau VM | vmbr5 (192.168.60.0/24) — Management/Réplication | Interne |
| Réseau App | vmbr6 (192.168.61.0/24) — Couche applicative | Interne |

### 5.2 VMs du cluster PostgreSQL

| VM | Rôle | Nœud | CPU | RAM | Disque | IP |
|----|------|------|-----|-----|--------|----|
| MSPR-DB-01 | Primary PostgreSQL 16 | pve-node1 | 2 vCPU | 3 Go | 30 Go | 192.168.60.11 |
| MSPR-DB-02 | Replica PostgreSQL 16 | pve-node2 | 2 vCPU | 2 Go | 30 Go | 192.168.60.12 |
| MSPR-MGT | Monitoring + Patroni + etcd | pve-node1 | 1 vCPU | 1 Go | 10 Go | 192.168.60.10 |

### 5.3 Architecture réseau

```
┌──────────────────────────────────────────────────────────┐
│              PROXMOX CLUSTER (pve-node1 + pve-node2)     │
│                                                          │
│  ┌─────────────── vmbr5: 192.168.60.0/24 ─────────────┐ │
│  │                                                      │ │
│  │  [MSPR-MGT]         [MSPR-DB-01]    [MSPR-DB-02]   │ │
│  │  192.168.60.10      192.168.60.11   192.168.60.12   │ │
│  │  Patroni/etcd       PRIMARY PG16    REPLICA PG16    │ │
│  │  pve-node1          pve-node1       pve-node2       │ │
│  │       │                  │◄──WAL Streaming──►│       │ │
│  │       └──────────────────┴───────────────────┘       │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                          │
│  ┌─────────────── vmbr6: 192.168.61.0/24 ─────────────┐ │
│  │         (Réservée couche applicative — P2)           │ │
│  └──────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────┘
         │
    [VPN WireGuard]
         │
    Accès administrateur distant (192.168.60.254)
```

---

## 6. Politique d'accès à la base de données

### 6.1 Principe de moindre privilège

| Rôle PostgreSQL | Droits | Utilisateurs associés |
|----------------|--------|----------------------|
| `wms_admin` | Superuser limité, DDL, VACUUM | DBA uniquement |
| `wms_app` | SELECT, INSERT, UPDATE, DELETE sur schéma `wms` | Application WMS |
| `wms_readonly` | SELECT uniquement sur toutes les tables | Reporting, supervision |
| `repuser` | REPLICATION uniquement | Replica DB-02 |
| `backup_user` | pg_read_all_data, pg_checkpoint | Scripts de sauvegarde |

### 6.2 Création des rôles

```sql
-- Rôle application
CREATE ROLE wms_app WITH LOGIN PASSWORD '***' NOSUPERUSER NOCREATEDB;
GRANT CONNECT ON DATABASE wms TO wms_app;
GRANT USAGE ON SCHEMA wms TO wms_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA wms TO wms_app;
GRANT USAGE ON ALL SEQUENCES IN SCHEMA wms TO wms_app;
ALTER DEFAULT PRIVILEGES IN SCHEMA wms
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO wms_app;

-- Rôle lecture seule
CREATE ROLE wms_readonly WITH LOGIN PASSWORD '***' NOSUPERUSER NOCREATEDB;
GRANT CONNECT ON DATABASE wms TO wms_readonly;
GRANT USAGE ON SCHEMA wms TO wms_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA wms TO wms_readonly;

-- Row Level Security (séparation par client)
ALTER TABLE article ENABLE ROW LEVEL SECURITY;
CREATE POLICY client_isolation ON article
    USING (id_client = current_setting('app.current_client_id')::INT);
```

### 6.3 pg_hba.conf (politique d'authentification)

```
# Connexions locales (maintenance)
local   all             postgres                        peer
local   all             wms_admin                       peer

# Réseau interne — application
host    wms             wms_app         192.168.61.0/24 scram-sha-256
host    wms             wms_readonly    192.168.60.0/24 scram-sha-256

# Réplication
host    replication     repuser         192.168.60.12/32 scram-sha-256
host    replication     repuser         192.168.60.10/32 scram-sha-256

# Administration DBA (réseau management uniquement)
host    all             wms_admin       192.168.60.0/24 scram-sha-256
```

---

## 7. Politique de sauvegarde

### 7.1 Stratégie 3-2-1

| Niveau | Type | Fréquence | Rétention | Destination |
|--------|------|-----------|-----------|-------------|
| L1 | Sauvegarde complète (pg_dump) | Quotidienne (00h00) | 7 jours | /backups/local |
| L2 | Sauvegarde incrémentale WAL | Continue (archivage WAL) | 15 min (RPO) | /backups/wal |
| L3 | pg_basebackup complet | Hebdomadaire (dimanche 02h00) | 30 jours | NAS externe |
| L4 | Export mensuel chiffré | 1er du mois | 90 jours | NAS offsite/Azure |

### 7.2 Objectifs

- **RPO :** 15 minutes (archivage WAL continu)
- **RTO :** 1 heure (failover Patroni ou restauration pg_basebackup)
- **Test restauration :** hebdomadaire automatisé (script test_restore.sh)

### 7.3 Répertoires

```
/backups/
├── postgresql/           # Dumps pg_dump quotidiens
├── wal/                  # Archives WAL (PITR)
├── basebackup/           # pg_basebackup hebdomadaires
└── logs/                 # Logs des scripts de sauvegarde
```
