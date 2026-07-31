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
    ├── scoring.py           # Scoring des AO via API OpenAI
    └── .env                 # Config BDD + clé API (non versionné)
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
- Coût API total : quelques centimes (gpt-4o-mini)

## Prochaines étapes

- [ ] Génération automatique de fiches techniques (à partir de la table `produits`)
- [ ] Génération automatique de devis (produits + client)
- [ ] Pré-filtrage par mots-clés au niveau du scraper (réduire le volume brut)
- [ ] Automatisation de l'exécution périodique du scraper (planification quotidienne/hebdomadaire)
- [ ] Intégration avec le module frontend (API REST à construire côté backend)

## Stack technique

- **Python** (Playwright pour le scraping, psycopg2 pour PostgreSQL)
- **PostgreSQL** (base de données commune aux 3 modules)
- **API OpenAI** (`gpt-4o-mini`) pour le scoring IA

---
*Stage Ecodelta 2026 — Pôle IA / Données produits*
