# Observatoire du Marché Data France

Pipeline d'ingestion et de modélisation des offres d'emploi *data* en France, à
partir de l'API **Offres d'emploi v2 de France Travail**. Projet portfolio
d'**Analytics Engineer** : de l'extraction brute (Python) jusqu'à la
modélisation analytique (dbt).

> **Statut** — Phase 1 (ingestion) terminée. La couche de transformation dbt
> est la suite prévue.

---

## Objectif

Constituer un jeu de données propre et reproductible sur le marché de l'emploi
data en France, pour répondre à des questions comme : quels métiers recrutent le
plus, quelles technologies sont demandées, quelle répartition géographique, etc.

L'accent est mis sur une chaîne **claire, traçable et honnête sur ses limites** —
plus que sur le volume : chaque choix (périmètre, dédoublonnage, non-modification
du brut) est documenté dans le code.

## Principe directeur : *raw jamais modifié*

Les offres extraites de l'API sont déposées **telles quelles** dans `data/raw/`.
Aucun filtrage, aucun dédoublonnage, aucun enrichissement n'est appliqué côté
Python. Toute transformation relève de la couche **dbt** en aval. Ce découplage
garantit qu'on peut rejouer et auditer les transformations sans jamais re-solliciter
l'API.

## Stratégie de périmètre : hybride `codeROME` / `motsCles`

L'exploration du référentiel ROME (cf. `exploration/`) a montré que les métiers
data ne se laissent pas capturer par une seule méthode :

| Méthode | Quand | Exemples |
|---|---|---|
| `codeROME` entier | métier ROME dédié à la data | M1405 (Data scientist), M1811 (Data engineer) |
| `motsCles` ciblé | intitulé isolé dans un code ROME fourre-tout | « data analyst », « data architect », « décisionnel », « business intelligence » |

Les codes fourre-tout (M1403, M1805, M1806, M1868) mélangent la data avec des
dizaines de métiers sans rapport : les prendre en entier polluerait le jeu de
données, d'où le recours au filtrage par mot-clé pour ces cas.

## Structure du projet

```
.
├── auth.py                 # Authentification OAuth2 (flux client_credentials)
├── search.py               # Recherche + pagination (header Range, plafond 1150)
├── pull_complet.py         # Pull de production : parcourt le périmètre → data/raw/
├── exploration/            # Scripts jetables ayant servi à cadrer le périmètre
│   ├── check_codeROME.py   #   intitulés d'un code ROME donné
│   ├── check_rome.py       #   répartition des codes ROME derrière un mot-clé
│   └── get_referentiel.py  #   pré-filtrage des appellations "data" du référentiel
├── data/raw/               # Dépôt brut horodaté (JSON), non versionné
└── docs/                   # Spec + comptes rendus de session
```

## Installation

```bash
pip install requests python-dotenv
```

Créer un fichier `.env` à la racine (jamais commité, cf. `.gitignore`) avec les
identifiants obtenus sur l'espace partenaire France Travail :

```
FT_CLIENT_ID=xxxxxxxx
FT_CLIENT_SECRET=xxxxxxxx
```

## Utilisation

```bash
# Vérifier que l'authentification fonctionne
python auth.py

# Lancer le pull complet du périmètre data → data/raw/offres_<horodatage>.json
python pull_complet.py
```

Chaque exécution produit un fichier horodaté distinct : rien n'est écrasé,
l'historique des pulls est conservé.

## Anatomie du fichier produit

```json
{
  "metadata": {
    "date_extraction": "...",
    "categories": [ { "nom": "...", "total_recupere": 0, "doublons_internes": 0 } ],
    "total_offres_brutes": 0,
    "total_offres_id_uniques": 0
  },
  "resultats": [ /* offres brutes, non modifiées */ ]
}
```

Le bloc `metadata` porte les mesures de qualité (volume par catégorie, doublons)
sans jamais toucher aux offres elles-mêmes.

## Exemple de run (17/07/2026)

| Catégorie | Filtre | Offres |
|---|---|---:|
| Data scientist | `codeROME=M1405` | 125 |
| Data engineer | `codeROME=M1811` | 450 |
| Data analyst | `motsCles=data analyst` | 300 |
| Data architect | `motsCles=data architect` | 45 |
| Décisionnel | `motsCles=décisionnel` | 85 |
| Business Intelligence | `motsCles=business intelligence` | 89 |
| **Total brut** | | **1 094** |
| **ID uniques** | | **552** |

L'écart entre offres brutes (1 094) et ID uniques (552) est **attendu** : une même
offre peut matcher plusieurs mots-clés ou codes ROME. Le dédoublonnage est traité
en aval par dbt (`stg_ft_offres`), pas à l'ingestion.

## Limites connues

- **Plafond API** : 1 150 résultats par recherche (pagination `Range` 0–1149).
  Au-delà, il faudrait affiner (ex. par date). Hors scope pour l'instant.
- **Index temps réel** : la pagination s'effectue sur un index vivant, d'où des
  doublons possibles au sein d'une même catégorie — mesurés, non corrigés à ce stade.
- **Traçabilité `motsCles`** : les offres remontées par mot-clé ne conservent pas,
  dans le JSON brut, la trace du mot-clé qui les a fait matcher (contrairement au
  champ natif `romeCode`). Choix assumé pour ne pas modifier le brut.

## Suite prévue

- Couche **dbt** : staging (`stg_ft_offres`, dédoublonnage), modèles intermédiaires
  et marts analytiques.
- Analyses : top technologies, répartition géographique, séries temporelles des
  volumes de recrutement.

---

## Stack

Python (requests, python-dotenv) · API France Travail Offres d'emploi v2 · dbt (à venir)
