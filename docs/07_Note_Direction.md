# Note à l'attention du Comité de Direction
## NordTransit Logistics — Sécurisation de la Base de Données WMS
**Destinataires :** Direction Générale, DAF, Responsable IT
**Date :** Juin 2026 | **Classification :** Confidentiel Direction
**Auteur :** Équipe Projet MSPR — Quentin GRENIER (Gestion de Projet)

---

## Objet : Risques cyber liés à la base de données WMS — Impact métier et mesures proposées

---

## 1. Résumé exécutif

La base de données WMS (Warehouse Management System) est le **cœur opérationnel de NordTransit Logistics**. Son indisponibilité, même partielle, provoque l'arrêt immédiat des activités de réception et d'expédition sur les quatre sites (Lens, Valenciennes, Arras, Cross-dock). Avec un chiffre d'affaires dépendant à 100% de la fluidité logistique, chaque heure d'interruption représente un risque économique direct.

Notre équipe a audité l'infrastructure existante et identifié **trois risques majeurs**, pour lesquels nous proposons des mesures concrètes, déjà partiellement mises en œuvre.

---

## 2. Risques identifiés et impact métier

### 🔴 RISQUE N°1 — Indisponibilité de la base de données (CRITIQUE)

**Situation actuelle :**
La base WMS repose sur une seule machine virtuelle MySQL sans réplication ni mécanisme de basculement automatique. Un simple incident matériel ou logiciel suffit à bloquer l'ensemble des opérations.

**Impact métier concret :**
- Blocage immédiat de 180 opérateurs entrepôt sur 4 sites
- Impossibilité de réceptionner les marchandises fournisseurs
- Impossibilité d'expédier les commandes clients
- Blocage des terminaux radiofréquence (scanners Wi-Fi)
- Interruption des flux EDI avec les transporteurs partenaires

**Impact financier estimé :**
- Entre 5 000 € et 15 000 € de pertes par heure d'arrêt (estimation conservative)
- Risque de pénalités contractuelles clients
- Atteinte à la réputation commerciale

**Mesure proposée (déjà mise en œuvre) :**
Déploiement d'une architecture de haute disponibilité :
- Base principale (DB-01) sur serveur Proxmox primaire
- Base secondaire synchronisée en temps réel (DB-02) sur serveur distinct
- Basculement automatique en moins de 30 minutes
- **RTO : 1 heure maximum | RPO : 15 minutes maximum**

---

### 🔴 RISQUE N°2 — Perte ou corruption des données (CRITIQUE)

**Situation actuelle :**
Les sauvegardes s'appuient sur des scripts sans campagne de restauration planifiée ni objectifs formalisés. Aucun test de restauration régulier n'est effectué. Une sauvegarde non testée est une fausse sécurité.

**Impact métier concret :**
- Perte de l'historique des mouvements de stock
- Impossibilité de reconstituer les entrées/sorties pour la facturation
- Perte de la traçabilité réglementaire des marchandises
- Risque de litige client en cas de perte de commandes

**Mesure proposée (déjà mise en œuvre) :**
- Sauvegardes automatisées quotidiennes avec vérification d'intégrité
- Test de restauration automatique chaque semaine
- Rétention à trois niveaux : 7 jours / 30 jours / 90 jours
- Capacité de restauration à un instant précis (perte maximale : 15 minutes)

---

### 🟠 RISQUE N°3 — Accès non autorisé aux données (HAUT)

**Situation actuelle :**
L'accès à la base de données n'est pas cloisonné. Un compte unique avec des privilèges étendus est utilisé par plusieurs applications et utilisateurs. En cas de compromission d'un poste ou d'un mot de passe, l'ensemble des données WMS est exposé.

**Impact métier concret :**
- Fuite de données clients et commerciales confidentielles
- Risque de manipulation frauduleuse des stocks
- Obligation légale RGPD : notification à la CNIL et aux clients concernés
- Atteinte à la confiance des partenaires et donneurs d'ordre

**Mesure proposée :**
- Séparation des accès par fonction (lecture seule / application / administration)
- Principe de moindre privilège : chaque compte n'accède qu'aux données nécessaires
- Authentification renforcée pour les accès administrateurs
- Isolation réseau : la base de données n'est accessible que depuis le réseau interne dédié (non exposée sur internet)

---

## 3. Synthèse des mesures et priorisation

| Priorité | Mesure | Bénéfice | Statut |
|----------|--------|----------|--------|
| 🔴 **1 — URGENT** | Architecture haute disponibilité (réplication + basculement) | RTO 1h / RPO 15 min | ✅ **Réalisé** |
| 🔴 **2 — URGENT** | Sauvegardes automatisées + tests restauration | Zéro perte de données testée | ✅ **Réalisé** |
| 🟠 **3 — IMPORTANT** | Cloisonnement des accès (moindre privilège) | Réduction surface d'attaque | ✅ **Réalisé** |
| 🟡 **4 — PLANIFIÉ** | Monitoring et alertes temps réel | Détection proactive des incidents | ✅ **Réalisé** |
| 🟡 **5 — PLANIFIÉ** | Externalisation des sauvegardes (NAS + cloud Azure) | Protection contre sinistre site | 🔄 **En cours** |

---

## 4. Investissements et retour sur investissement

Les mesures mises en place s'appuient principalement sur l'infrastructure Proxmox existante (Dell PowerEdge R630, NAS). Les coûts additionnels sont limités :

| Poste | Coût estimé | Bénéfice |
|-------|------------|---------|
| Logiciels (PostgreSQL, Patroni, etcd) | **0 €** (open-source) | Architecture HA complète |
| Stockage NAS (sauvegardes supplémentaires) | ~500 €/an | Rétention 90 jours |
| Espace Azure (copies offsite) | ~200 €/an | Protection sinistre majeur |
| Formation équipe IT (2 jours) | ~1 200 € | Maîtrise des procédures PRA |
| **Total annuel** | **~1 900 €/an** | **Évite 5 000–15 000 €/heure d'arrêt** |

**Le retour sur investissement est atteint dès la première heure d'arrêt évitée.**

---

## 5. Recommandations à la direction

Nous recommandons à la direction de NordTransit Logistics de :

1. **Valider et pérenniser l'architecture HA** mise en place — elle doit être considérée comme une infrastructure de production permanente, pas un projet ponctuel.

2. **Budgéter les sauvegardes externalisées** (Azure ou prestataire) pour protéger NTL contre un sinistre sur le site de Lille (incendie, inondation, vol).

3. **Formaliser l'astreinte technique** pour la BDD : le système WMS fonctionne de 5h30 à 18h30 ; un intervenant joignable en moins de 30 minutes est indispensable.

4. **Planifier un exercice PRA semestriel** (simulation de panne) pour maintenir les compétences de l'équipe et valider les procédures documentées.

5. **Étendre l'authentification multifacteur** (actuellement limitée à l'équipe IT) aux administrateurs de bases de données et aux accès distants sensibles.

---

*Cette note a été rédigée par l'équipe projet dans le cadre de la mission MSPR 2025-2026. Elle est destinée à un usage interne et ne constitue pas un audit de sécurité exhaustif.*
