# Projet Ecodelta — Module Veille IA des Appels d'Offres

Module développé dans le cadre du stage Ecodelta (2026), pôle **IA / Données produits**.

## Objectif

Automatiser la veille des appels d'offres marocains : scraper les AO publiés sur les sites officiels, les stocker en base, puis utiliser l'IA pour identifier automatiquement ceux qui sont pertinents pour l'activité d'Ecodelta.

## Profil Ecodelta (utilisé pour le scoring)

Ecodelta est spécialisée dans :
- Énergie solaire (injection solaire, pompage solaire)
- Sécurité périmétrique et contrôle d'accès
- Gestion de parking
- Affichage dynamique (murs d'image)
- Effaroucheurs laser/acoustique
- Construction de terrains de padel

Secteurs clients ciblés : industrie, hôtellerie/tourisme, ports, aéroports, secteur public.

## Architecture

```
ProjetEcodelta/
├── ecodelta_backend/       # Scraping des appels d'offres
│   ├── db.py               # Connexion PostgreSQL
│   ├── scraper_ao.py       # Scraper marchespublics.gov.ma
│   └── .env                # Config BDD (non versionné)
│
└── ecodelta_ai/             # Traitement IA
    ├── db.py                # Connexion PostgreSQL (copie)
    ├── scoring.py            # Scoring des AO via API OpenAI
    ├── fiches_techniques.py  # Génération de fiches techniques produits
    ├── devis.py              # Génération de devis (client + produits)
    └── .env                  # Config BDD + clé API (non versionné)
```

## Base de données (PostgreSQL — `ecodelta_db`)

| Table | Rôle |
|---|---|
| `appels_offres` | AO scrapés + score IA + justification |
| `clients` | Suivi client (module CRM) |
| `produits` | Catalogue produits/prix Ecodelta |
| `devis` | Devis générés (module génération) |

Schéma complet : voir les scripts SQL de création dans `ecodelta_backend/schema.sql` (à ajouter si besoin).

## 1. Scraper (`ecodelta_backend/scraper_ao.py`)

Récupère les appels d'offres depuis **marchespublics.gov.ma**.

**Fonctionnement :**
1. Navigue vers la recherche avancée
2. Lance une recherche large (tous critères)
3. Affiche 500 résultats par page
4. Parcourt automatiquement toutes les pages (pagination)
5. Parse chaque ligne (référence, titre, acheteur, lieu, date limite)
6. Insère en base avec anti-doublons (vérification sur la référence)

**Lancer :**
```bash
cd ecodelta_backend
venv\Scripts\activate
python scraper_ao.py
```

**Résultat attendu :** insertion des nouveaux AO, les doublons déjà en base sont ignorés automatiquement.

## 2. Scoring IA (`ecodelta_ai/scoring.py`)

Évalue chaque AO non encore scoré via l'API OpenAI (`gpt-4o-mini`), selon le vrai profil métier d'Ecodelta.

**Fonctionnement :**
- Récupère les AO où `score_ia IS NULL`
- Envoie chacun à l'IA avec un prompt détaillant précisément ce qu'Ecodelta fait / ne fait pas
- Reçoit un score de 0 à 10 + une justification
- Enregistre directement en base (`score_ia`, `justification_ia`)
- Traite par lots (500 par défaut) avec confirmation entre chaque lot
- Affiche les tokens consommés et le coût estimé après chaque lot

**Grille de notation :**
- 8-10 : correspond directement à un domaine d'expertise Ecodelta
- 4-7 : lien indirect possible
- 0-3 : aucun rapport

**Lancer :**
```bash
cd ecodelta_ai
venv\Scripts\activate
python scoring.py
```

**Voir les AO les plus pertinents (SQL) :**
```sql
SELECT titre, score_ia, justification_ia, date_limite, lien
FROM appels_offres
WHERE score_ia >= 7
ORDER BY score_ia DESC;
```

## 3. Fiches techniques (`ecodelta_ai/fiches_techniques.py`)

Génère automatiquement une fiche technique commerciale pour chaque produit du catalogue, via l'API OpenAI.

**Fonctionnement :**
- Récupère les produits où `fiche_technique IS NULL`
- Génère : titre accrocheur, présentation orientée bénéfices, caractéristiques techniques, applications recommandées (2-3 secteurs max, sélectionnés et justifiés selon le produit — pas une liste générique des 5 secteurs à chaque fois), prix indicatif
- Stocke le résultat en JSON dans la colonne `fiche_technique` (type `JSONB`)

**Lancer :**
```bash
cd ecodelta_ai
python fiches_techniques.py
```

**Prérequis BDD (une seule fois) :**
```sql
ALTER TABLE produits ADD COLUMN fiche_technique JSONB;
```

## 4. Génération de devis (`ecodelta_ai/devis.py`)

Combine un client + une liste de produits/quantités pour générer un devis.

**Fonctionnement :**
- Les **calculs de prix sont faits en Python** (fiable, aucun risque d'erreur de calcul par l'IA)
- L'IA rédige uniquement le texte d'introduction/conclusion, personnalisé selon le client
- Le devis est enregistré en base avec `statut = "brouillon"` et `valide_par_humain = False`

**Lancer (exemple) :**
```python
from devis import generer_devis
generer_devis(client_id=1, produits_quantites=[(1, 2), (4, 1)])
```

**Règle de sécurité (conforme au brief) :** un devis généré par l'IA n'est jamais envoyé directement au client — il reste en statut `brouillon` jusqu'à validation humaine explicite (mise à jour manuelle de `valide_par_humain`).

## ⚠️ Données actuelles : fictives / de test

Les tables `produits` et `clients` contiennent actuellement des **données de test**, construites à partir des informations publiques du site ecodelta.ma — **pas le vrai catalogue produits ni les vrais clients de l'entreprise**.

Avant mise en production réelle, il faut :
1. Récupérer le vrai catalogue produits/prix auprès du référent Ecodelta
2. Récupérer une vraie liste de clients/prospects
3. Vider les tables de test et importer les vraies données (`DELETE FROM produits; DELETE FROM clients;` puis import)

Le code des 3 scripts IA (`scoring.py`, `fiches_techniques.py`, `devis.py`) n'a besoin d'aucune modification pour fonctionner avec les vraies données — seul le contenu des tables change.

## Configuration (`.env`)

Chaque dossier a son propre `.env` (non versionné, à créer localement) :

```
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecodelta_db
DB_USER=postgres
DB_PASSWORD=xxx

# Uniquement dans ecodelta_ai
OPENAI_API_KEY=xxx
```

⚠️ Ne jamais committer `.env` — protégé par `.gitignore`.

## Résultats (dernier passage complet)

- **~3900 AO** scrapés depuis marchespublics.gov.ma
- **37 AO** identifiés comme réellement pertinents (score ≥ 7)
- **9 fiches techniques** générées (catalogue de test)
- **1 devis** généré et testé (pipeline validé)
- Coût API total cumulé (scoring + fiches + devis) : quelques centimes (gpt-4o-mini)

## Prochaines étapes

- [x] Scoring IA des appels d'offres
- [x] Génération de fiches techniques
- [x] Génération de devis (pipeline de base)
- [ ] Remplacer les données de test par le vrai catalogue produits et les vrais clients Ecodelta
- [ ] Ajouter la TVA / conditions de paiement dans le calcul des devis
- [ ] Pré-filtrage par mots-clés au niveau du scraper (réduire le volume brut)
- [ ] Automatisation de l'exécution périodique du scraper (planification quotidienne/hebdomadaire)
- [ ] Intégration avec le module frontend (API REST à construire côté backend)
- [ ] Export PDF des fiches techniques et devis (actuellement JSON/texte uniquement)

## Stack technique

- **Python** (Playwright pour le scraping, psycopg2 pour PostgreSQL)
- **PostgreSQL** (base de données commune aux 3 modules)
- **API OpenAI** (`gpt-4o-mini`) pour le scoring IA

---
*Stage Ecodelta 2026 — Pôle IA / Données produits*