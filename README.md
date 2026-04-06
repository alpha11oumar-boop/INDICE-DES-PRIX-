# Indices de prix immobiliers hédoniques — Hauts-de-France (2014–2025)

> Master 1 Statistique & Économétrie — Université de Strasbourg  
> UE Conduite de Projet — 2025–2026

---
## Auteurs

| Nom | GitHub |
|-----|--------|
| DIB Adel | [@adeldib](https://github.com/adeldib) |
| BAH Alpha Oumar | [@alpha11oumar-boop](https://github.com/alpha11oumar-boop) |
— Master 1 APE / SE, Université de Strasbourg*

---

## Objectif

Construire des **indices de prix immobiliers à qualité constante** au niveau **commune × mois** pour la région Hauts-de-France sur la période 2010–2025, en distinguant les **appartements** et les **maisons**.

Le prix moyen observé varie lorsque la composition des biens vendus change (plus de neuf, de grandes surfaces, de biens mieux situés), sans que le marché lui-même n'ait évolué. Le modèle hédonique corrige ce biais de composition.

---

## Données

| Source | Période | Observations | Format |
|--------|---------|-------------|--------|
| **DVF+** | 2014–2025 | 1 181 643 | CSV (séparateur `\|`) |
| **DV3F** | 2010–2013 | 236 646 | Parquet |

- **Téléchargement DVF+** : [data.gouv.fr – DVF+](https://www.data.gouv.fr/fr/datasets/demandes-de-valeurs-foncieres-geolocalisees/) → sélectionner la région Hauts-de-France, format CSV.
- **DV3F 2010–2013** : fourni par l'enseignant via Moodle.
- **Région couverte** : départements 02 (Aisne), 59 (Nord), 60 (Oise), 62 (Pas-de-Calais), 80 (Somme).

**Après nettoyage** : 1 074 908 ventes effectives retenues (filtre 200–20 000 €/m²).

---

## Méthodologie

### Modèle hédonique log-linéaire

Deux modèles sont estimés séparément (appartements / maisons) par OLS via le package `fixest` :

```
log(pᵢ) = α + xᵢ′β + γ_{t(i)} + δ_{d(i)} + μ_{c(i)} + θ_{d(i),a(i)} + εᵢ
```

| Terme | Rôle |
|-------|------|
| `xᵢ′β` | Variables de qualité : log(surface), VEFA, typologies T1/T3/T4/T5 (T2 = référence) |
| `γ_t` | Effet fixe temps (mois × année) — tendance du marché à qualité constante |
| `δ_d` | Effet fixe département |
| `μ_c` | Effet fixe commune (non estimé pour les 100 communes les moins liquides) |
| `θ_{d,a}` | Interaction département × année |

### Choix des variables de qualité

Les variables incluses dans `xᵢ′β` ont été choisies pour neutraliser les effets de composition les plus importants tout en restant contraintes par ce que DVF+ et DV3F fournissent.

**`log(surface bâtie)`** — Le log est préféré au niveau pour capturer la relation non linéaire entre surface et prix/m² : les petites surfaces se vendent structurellement plus cher au mètre carré (rendements décroissants). Utiliser le niveau biaiserait l'indice en faveur des périodes où l'on vend beaucoup de grands logements.

**`VEFA`** (Vente en l'État Futur d'Achèvement) — Le neuf se vend avec une prime systématique sur l'ancien, indépendamment de la localisation. Sans ce contrôle, un pic de construction neuve dans une commune ferait artificiellement monter son indice.

**Typologies T1–T5 (appartements), P1–P5 (maisons)** — Les F1 et F5 ont des prix/m² structurellement différents des F2–F3 (références). Cette correction est cruciale car le mix de typologies vendues varie selon les cycles économiques et les marchés locaux. T2 est choisi comme référence car c'est la catégorie la plus fréquente (54 002 observations), ce qui stabilise l'estimation.

**`log(surface terrain)` — maisons uniquement** — Le terrain est une composante de valeur propre aux maisons, absente pour les appartements. Ne pas l'inclure ferait confondre la valeur du foncier avec la tendance de marché dans les communes périurbaines.

**Variables non incluses et pourquoi** — L'état du bien, l'étage, l'exposition et la présence d'un garage ne sont pas disponibles dans DVF+. Leur absence est une limite connue (voir section Limites), mais elle est subie et non choisie.

### Pipeline en 7 étapes

1. **Estimation OLS** du modèle hédonique sur l'ensemble des données.
2. **Prix net à qualité constante** : `log P^net_i = ŷᵢ − Σ β̂_k · x_ik` (on soustrait la contribution des variables de qualité).
3. **Correction des petites communes** : les 100 communes les moins liquides reçoivent une correction par moyenne des résidus plutôt qu'un effet fixe direct.
4. **Agrégation commune × mois** : médiane du log prix net (seuil minimum : 3 transactions).
5. **Complétion des périodes manquantes** : prédiction via les coefficients estimés pour les mois sans transaction, avec garde-fous (±3 écarts-types, tous effets fixes disponibles requis).
6. **Indice base 100** : `Index_{c,t} = 100 × exp(log P^net_{c,t} − log P^net_{c,t₀})` avec une période de référence t₀ commune à toutes les communes.
7. **Garde-fou final** : `|diff_log| ≤ log(5)` pour éliminer les variations aberrantes.

> **Note** : l'objectif du modèle est la prédiction, non l'inférence causale. La multicolinéarité entre dummies n'est pas un problème ici.

---

## Résultats

### Appartements

| Paramètre | Estimate | Std. Error | Signif. |
|-----------|----------|-----------|---------|
| `log_sbati` | −0.2320 | 0.0060 | * |
| `vefa_num` | +0.3569 | 0.0398 | * |
| `T1` | −0.0214 | 0.0055 | * |
| `T3` | +0.0143 | 0.0047 | ** |
| `T4` | −0.0225 | 0.0063 | * |
| `T5` | +0.0116 | 0.0094 | ns |

**Qualité du modèle** : 38 282 observations · 430 communes · R² ajusté = 0.505 · RMSE = 0.302  
**Indice final** : Min = 86.3 · Médiane = 94.6 · Moyenne = 94.6 · Max = 100.1

**Interprétations clés** :
- **Prime VEFA** : +100 × (exp(0.357) − 1) ≈ **+42,9 %** par rapport à un bien ancien comparable.
- **Élasticité surface** : −0.232 — les petits appartements se vendent plus cher au m², cohérent avec les économies d'échelle documentées dans la littérature.
- **Hétérogénéité spatiale** : Lille surperforme la moyenne régionale sur toute la période ; Roubaix présente un phénomène de rattrapage ; Arras suit le cycle régional.

### Maisons

| Paramètre | Estimate | Std. Error | Signif. |
|-----------|----------|-----------|---------|
| `log_sbati` | −0.1657 | 0.0012 | * |
| `vefa_num` | +0.2108 | 0.0910 | * |

**Qualité du modèle** : 806 942 observations · 4 192 communes · R² ajusté = 0.425 · RMSE = 0.395  
**Indice final** : Min = 78.4 · Médiane = 88.9 · Moyenne = 90.8 · Max = 109.1

**Interprétations clés** :
- **Prime VEFA maisons** : +100 × (exp(0.211) − 1) ≈ **+23,5 %** — inférieure à celle des appartements, les maisons neuves étant moins concentrées en zone dense.
- **R² plus faible** (0.425 vs 0.505) : attendu, en raison de la plus grande hétérogénéité des maisons (terrain, configuration, état) non captée par les variables disponibles.
- **Post-Covid 2020–2022** : surperformance des maisons liée à l'effet de « fuite vers l'espace ».

### Régimes temporels identifiés (2010–2025)

| Période | Dynamique |
|---------|-----------|
| 2010–2013 | Marché post-crise 2008 · stagnation / légère baisse |
| 2014–2019 | Stabilisation puis reprise modérée portée par la baisse des taux |
| 2020–2022 | Accélération post-Covid · préférences résidentielles et taux bas |
| 2023–2025 | Retournement consécutif à la remontée des taux BCE |

---

## Visualisations

### Courbes temporelles

Les courbes représentent l'évolution de `Index_{c,t}` (base 100 = janvier 2014) pour un sélection de communes — Lille, Roubaix, Arras, Amiens, Beauvais — ainsi qu'un **benchmark régional** calculé comme la médiane de l'indice sur l'ensemble des communes à chaque période.

Ces graphiques permettent de lire trois choses simultanément : le **niveau** de départ de chaque commune (son positionnement relatif en 2014), la **pente** de croissance sur la période, et les **ruptures** conjoncturelles (Covid, remontée des taux). Une commune dont la courbe dépasse le benchmark régional connaît une appréciation plus rapide que la moyenne des Hauts-de-France à qualité constante.

Un graphique supplémentaire superpose l'indice appartements et maisons pour une même commune (ici Lille), ce qui permet de visualiser les divergences de dynamique entre les deux segments sur un même marché local.

### Cartes choroplèthes

Deux cartes sont produites sur le fond de carte des communes des Hauts-de-France (source GISCO/Eurostat via `giscoR`).

**Carte 1 — Indice moyen** : pour chaque commune, on calcule la moyenne de `Index_{c,t}` sur l'ensemble de la période, en ne retenant que les observations réelles (pas les périodes complétées par prédiction, avec un minimum de 5 observations). Une valeur > 100 signifie que la commune a en moyenne connu une appréciation depuis 2014 ; < 100 une dépréciation. La discrétisation en quantiles permet de visualiser la distribution spatiale sans que les extrêmes n'écrasent la lecture.

**Carte 2 — Effets fixes communaux** : on représente `exp(μ_c)`, le niveau de prix relatif structurel de chaque commune estimé par le modèle, à qualité constante et indépendamment de la période. Contrairement à la carte 1 qui reflète une dynamique temporelle, cette carte reflète un **niveau absolu** : une commune avec `exp(μ_c) = 1.3` a structurellement des prix 30 % supérieurs à la commune de référence, toutes choses égales par ailleurs. C'est une mesure de l'attractivité résidentielle "pure", purifiée de la composition des biens et des tendances temporelles.

### Heatmap commune × temps

La heatmap représente les 30 communes les plus actives en volume de transactions, sur l'axe vertical, croisées avec les années sur l'axe horizontal. Chaque cellule est colorée selon la valeur annuelle médiane de `Index_{c,t}`.

Cette visualisation est complémentaire aux courbes : là où les courbes permettent de suivre une commune précise dans le temps, la heatmap permet de **comparer simultanément** le comportement de toutes les communes et de détecter des patterns collectifs — une colonne très claire signifie une année de hausse généralisée, une ligne qui s'assombrit progressivement signifie qu'une commune décroche progressivement de la dynamique régionale. Les ruptures comme 2020–2022 (Covid) ou 2023 (remontée des taux) apparaissent clairement comme des ruptures de couleur sur l'ensemble des colonnes.

---

## Structure du dépôt

```
indices-prix-immo-hdf/
├── README.md                          ← ce fichier
├── analyse_immo_hdf.Rmd               ← code principal (pipeline complet)
├── data/
│   └── README.md                      ← instructions de téléchargement des données
└── outputs/
    └── README.md                      ← description des fichiers produits
```

---

## Reproduire les résultats

### Prérequis R

```r
install.packages(c(
  "data.table",   # manipulation des grandes bases
  "arrow",        # lecture du format Parquet
  "fixest",       # OLS avec effets fixes (feols)
  "ggplot2",      # visualisations
  "sf",           # données spatiales
  "giscoR",       # fonds de carte GISCO/Eurostat
  "classInt",     # discrétisation pour cartes choroplèthes
  "scales",       # formatage des axes
  "tidyr"         # manipulation de données
))
```

### Étapes

1. Télécharger DVF+ pour les Hauts-de-France depuis data.gouv.fr (format CSV).
2. Récupérer DV3F 2010–2013 (Parquet) depuis Moodle.
3. Adapter les chemins dans le chunk `lecture_donnees` du fichier `.Rmd`.
4. Knitter le fichier : `rmarkdown::render("analyse_immo_hdf.Rmd")`.

> **Note sur la mémoire** : le modèle maisons porte sur ~807 000 observations avec ~4 200 effets fixes communaux. Prévoir au moins 8 Go de RAM disponible. Tester d'abord sur un sous-échantillon (`dt[sample(.N, 50000)]`) avant de lancer sur la base complète.

---

## Limites

- Variables de qualité manquantes dans DVF : état du bien, étage, exposition, surface du terrain.
- Spécification maisons restreinte (pas de typologies disponibles) → within R² = 0.023.
- La complétion des trous suppose la stabilité temporelle des effets fixes.
