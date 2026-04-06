# Indices de prix immobiliers hédoniques — Hauts-de-France (2010–2025)

> Master 1 Statistique & Économétrie — Université de Strasbourg  
> UE Conduite de Projet — 2025–2026

---

## Présentation

Ce projet construit des **indices de prix immobiliers à qualité constante** par commune et par mois pour la région Hauts-de-France, séparément pour les **appartements** et les **maisons**.

L'approche hédonique (modèle OLS en log-prix avec effets fixes) permet d'isoler l'évolution réelle du marché des effets de composition — surface, neuf/ancien, typologie des biens.

---

## Contenu du repo
```
.
├── indices_prix_immo.Rmd   # Code source complet
└── README.md
```

> Les données brutes (DVF+ et DV3F) et les outputs CSV ne sont pas inclus. Voir ci-dessous pour les obtenir.

---

## Données nécessaires

| Source | Période | Format | Lien |
|--------|---------|--------|------|
| DVF+ | 2014–2025 | CSV | [data.gouv.fr](https://www.data.gouv.fr/fr/datasets/demandes-de-valeurs-foncieres/) |
| DV3F | 2010–2013 | Parquet | [datafoncier.cerema.fr](https://datafoncier.cerema.fr/dv3f) |

Restreindre aux départements **02 · 59 · 60 · 62 · 80**, puis adapter les chemins en tête du `.Rmd` :
```r
CHEMIN_DVF  <- "chemin/vers/dvf_plus_hauts_de_france.csv"
CHEMIN_DV3F <- "chemin/vers/dv3f_hauts_de_france.parquet"
```

---

## Utilisation

### Packages requis
```r
install.packages(c("data.table", "arrow", "fixest", "ggplot2", "sf"))
```

### Lancer le rapport
```r
rmarkdown::render("indices_prix_immo.Rmd")
```

Le rendu produit un rapport HTML avec toutes les visualisations, et exporte deux fichiers :
- `indice_appartements_hdf.csv`
- `indice_maisons_hdf.csv`

---

## Méthode en bref

- Modèle OLS : `log(prix/m²) ~ log(surface) + VEFA + typologies | temps + département + commune + département×année`
- Estimation via [`fixest`](https://lrberge.github.io/fixest/)
- Indice base 100 = janvier 2014, par commune × mois
- Petites communes (100 moins actives) : effet fixe communal fixé à 0, recalage par résidus

---

## Auteurs


— Master 1 APE / SE, Université de Strasbourg*
