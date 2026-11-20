# cfadiag

**Diagnostics de fiabilité et de validité pour les modèles de mesure estimés avec lavaan.**

`cfadiag` rassemble dans un rapport en français les saturations standardisées, les coefficients de fiabilité, les indices de validité convergente et discriminante, les indices d’ajustement et les signaux à examiner. Il s’adresse aux chercheurs, aux enseignants et aux étudiants qui souhaitent documenter la qualité d’un modèle de mesure et comprendre les limites des scores qu’ils en tirent.

Le package reçoit un modèle déjà estimé avec `lavaan`. Il accompagne son interprétation : il ne choisit pas automatiquement les indicateurs, ne modifie pas le modèle et ne transforme pas des seuils conventionnels en verdicts de validité.

**Version documentée : 0.2.3 · Auteur : Daniel Coulombe · Licence : GPL-3**

## Fonctionnalités

- Rapport synthétique en français, avec sorties R réutilisables.
- Saturations entièrement standardisées et signaux de charges faibles, négatives, croisées ou hors des bornes usuelles.
- Oméga de la somme brute, fiabilité composite de la somme standardisée (CR), alpha et coefficient H.
- Variance moyenne extraite (AVE), variance partagée maximale (MSV), variance partagée moyenne (ASV) et tableau de Fornell–Larcker.
- HTMT arithmétique, HTMT2 géométrique et intervalles bootstrap percentile pour les données compatibles.
- Indices d’ajustement avec identification des versions standard ou corrigées disponibles.
- Prise en charge des matrices de covariance, des données manquantes traitées par FIML, des indicateurs ordinaux et des modèles multigroupes, selon les conditions précisées ci-dessous.
- Graphiques et simulation de données factorielles pour l’enseignement et l’exploration méthodologique.

## Installation

Le package nécessite **R ≥ 4.1.0** et **lavaan ≥ 0.6-12**. Il ne contient pas de code à compiler.

### Depuis GitHub

Après publication du dépôt, remplacez `VOTRE_IDENTIFIANT` par le compte ou l’organisation qui héberge `cfadiag` :

```r
install.packages("remotes")
remotes::install_github("dco67/cfadiag")
```

Pour une version étiquetée, ajoutez son tag, par exemple `@v0.2.3`, une fois ce tag publié.

### Depuis une archive source

Téléchargez l’archive dans votre dossier de travail, puis installez d’abord la dépendance si nécessaire :

```r
install.packages("lavaan")
install.packages("cfadiag_0.2.3.tar.gz", repos = NULL, type = "source")
```

`semTools` et `testthat` sont facultatifs pour les vérifications. `knitr` et `rmarkdown` servent à construire les vignettes; ils ne sont pas nécessaires pour calculer un diagnostic courant.

## Première utilisation

Cet exemple utilise les données classiques de Holzinger et Swineford, incluses dans `lavaan`. Il examine trois facteurs corrélés, chacun mesuré par trois indicateurs.

```r
library(lavaan)
library(cfadiag)

modele <- '
  visuel  =~ x1 + x2 + x3
  verbal  =~ x4 + x5 + x6
  vitesse =~ x7 + x8 + x9
'

fit <- cfa(
  modele,
  data = HolzingerSwineford1939,
  std.lv = TRUE
)

diagnostic <- cfadiag(fit)
diagnostic
```

Le rapport réunit l’ajustement, la fiabilité, les éléments de validité et les signaux du modèle. **Une solution convergée, un coefficient élevé ou l’absence de signal ne démontre pas à lui seul la validité du modèle.**

Les résultats restent accessibles séparément :

```r
diagnostic$fiabilite
diagnostic$validite$table
diagnostic$validite$fornell_larcker
diagnostic$htmt
diagnostic$ajustement
diagnostic$signaux
```

| Élément du rapport | Contenu |
|---|---|
| `apercu` | Groupe, nombre de facteurs et d’indicateurs, effectif et estimateur |
| `charges` | Matrice des saturations entièrement standardisées |
| `fiabilite` | Oméga, CR, alpha, H et statut des calculs par facteur |
| `validite` | Tableau AVE/MSV/ASV et matrice de Fornell–Larcker |
| `htmt` | Matrice HTMT, lorsqu’elle est applicable et calculable |
| `ajustement` | Indices, version utilisée, repères et comparaison descriptive |
| `signaux` | Points à examiner dans la solution |
| `htmt_boot` | Intervalles HTMT demandés explicitement; `NULL` par défaut |

## Bien définir le score et le dénominateur

Les coefficients du tableau de fiabilité n’ont pas tous la même cible :

| Coefficient | Cible du calcul |
|---|---|
| `omega` | Fiabilité de la somme des indicateurs dans leurs unités d’origine |
| `CR` | Fiabilité de la somme des indicateurs standardisés |
| `alpha` | Alpha calculé à partir de la covariance observée fournie au modèle |
| `H` | Coefficient associé à une combinaison à poids optimaux selon le modèle |

Oméga et CR peuvent donc différer lorsque les indicateurs n’ont pas les mêmes unités ou variances. Leurs dénominateurs incluent les covariances résiduelles représentées par le modèle. H utilise également la matrice résiduelle complète.

L’argument `variance` choisit le dénominateur de l’oméga et de la CR :

```r
# Variance de la somme reproduite par le modèle : choix par défaut.
d_impliquee <- cfadiag(fit, variance = "impliquee")

# Variance de la somme issue de la covariance observée.
d_observee <- cfadiag(fit, variance = "observee")

d_impliquee$fiabilite
d_observee$fiabilite
```

Les deux choix peuvent donner des résultats différents si le modèle ne reproduit pas exactement les covariances des indicateurs. Le choix `"observee"` correspond, pour l’oméga d’une somme à poids unitaires d’indicateurs continus, au dénominateur observé utilisé par défaut dans `semTools::compRelSEM()`.

**Les items inversés ne sont jamais recodés automatiquement.** Une saturation négative peut être cohérente avec le codage ou l’orientation du facteur. Il faut définir le score souhaité avant d’interpréter sa fiabilité.

## HTMT, HTMT2 et incertitude

Le HTMT utilise les corrélations issues de la **covariance observée**, et non celles que le modèle reproduit. Les valeurs absolues sont utilisées par défaut. Le HTMT2 s’obtient par la fonction dédiée sur un objet `cfaModel` :

```r
m <- as.cfaModel(fit)

compute_htmt(m, type = "htmt")
compute_htmt(m, type = "htmt2")
```

Pour ajouter des intervalles HTMT au rapport :

```r
d_boot <- cfadiag(
  fit,
  htmt_boot = TRUE,
  R = 1000,
  seed = 123
)

d_boot$htmt_boot
```

Pour demander spécifiquement des intervalles **HTMT2** ou contrôler le niveau de confiance :

```r
b_htmt2 <- boot_htmt(
  m,
  type = "htmt2",
  R = 1000,
  seed = 123,
  level = 0.95
)

b_htmt2$table
```

Les intervalles sont des intervalles percentile. Examinez leur largeur et `n_valides`, le nombre de réplications effectivement utilisables pour chaque paire. Une paire sans assez de réplications valides reçoit des bornes manquantes. Une graine explicite permet la reproduction du calcul tout en restaurant l’état aléatoire global à sa sortie.

Le bootstrap nécessite les données brutes et n’est pas proposé pour les indicateurs ordinaux dans cette version. Les paires partageant un indicateur, ayant trop peu d’indicateurs ou des corrélations non calculables peuvent produire `NA`. HTMT2 est notamment sensible aux corrélations nulles dans les moyennes géométriques.

## Adapter les repères descriptifs

Les repères par défaut sont CFI/TLI ≥ 0,95, RMSEA ≤ 0,06 et SRMR ≤ 0,08. Ils peuvent être adaptés au contexte :

```r
d_adapte <- cfadiag(
  fit,
  reperes = list(cfi = 0.90, tli = 0.90, rmsea = 0.08, srmr = 0.10),
  charge_faible = 0.50,
  htmt_alerte = 0.90
)
```

Ces valeurs servent à organiser l’examen du modèle. Un signal de charge faible ne prescrit pas la suppression d’un item; une covariance résiduelle appelle une justification de contenu ou de méthode. Pour un modèle juste identifié, les repères d’ajustement ne sont pas évalués comme un test informatif de sa structure.

La colonne `version` de `diagnostic$ajustement` identifie les indices standard, robustes ou corrigés retenus. Elle permet de distinguer des résultats qui portent le même nom mais n’utilisent pas la même correction.

## Données manquantes, indicateurs ordinaux et matrices

- **FIML.** Le diagnostic reprend la covariance observée estimée par `lavaan`. L’alpha correspondant n’est pas nécessairement l’alpha brut sur les seuls cas complets. Avec `missing = "auto"`, le bootstrap suit les métadonnées de l’estimation; sous FIML, la covariance est réestimée par EM à chaque réplication.
- **Indicateurs ordinaux.** Les calculs reposent sur les réponses latentes continues et les covariances ou corrélations catégorielles fournies par `lavaan`. En particulier, alpha peut être calculé sur la covariance polychorique : ces indices ne décrivent pas directement la fiabilité d’une somme des catégories observées.
- **Matrices de covariance.** Un modèle ajusté avec `sample.cov` et `sample.nobs` peut être diagnostiqué. Les opérations qui exigent les données brutes, dont le bootstrap, ne sont pas disponibles si ces données ne sont pas fournies.

Un exemple d’estimation avec réponses partielles :

```r
HS_incomplet <- HolzingerSwineford1939
HS_incomplet$x2[1:20] <- NA

fit_fiml <- cfa(
  modele,
  data = HS_incomplet,
  estimator = "MLR",
  missing = "fiml",
  std.lv = TRUE
)

cfadiag(fit_fiml)
```

La présence d’un calcul FIML ne démontre pas les hypothèses sur le mécanisme d’absence. Les conditions de recueil et la pertinence du modèle restent à discuter.

## Modèles structuraux et multigroupes

Pour les modèles structuraux, les corrélations utilisées dans les diagnostics sont les **corrélations impliquées entre facteurs**, et non les seules corrélations de leurs perturbations résiduelles. Les contraintes de la partie structurale peuvent néanmoins modifier ces relations; l’évaluation de la mesure doit en tenir compte.

Un modèle multigroupe produit un objet `cfadiag_multigroupe`, contenant un diagnostic par groupe :

```r
fit_groupes <- cfa(
  modele,
  data = HolzingerSwineford1939,
  group = "school"
)

d_groupes <- cfadiag(fit_groupes)
names(d_groupes)
d_groupes[["Pasteur"]]
```

Les saturations, les coefficients de fiabilité et les éléments de validité sont propres au groupe. **L’ajustement global reste celui du modèle entier, tous groupes confondus**, et le rapport le précise. Produire ces diagnostics ne constitue pas en soi un test d’invariance entre groupes.

## Graphiques

Quatre représentations sont disponibles pour un diagnostic individuel :

```r
plot(diagnostic, type = "charges")
plot(diagnostic, type = "fiabilite")
plot(diagnostic, type = "validite")
plot(diagnostic, type = "htmt")
```

Pour un modèle multigroupe, sélectionnez d’abord le diagnostic d’un groupe, par exemple `plot(d_groupes[["Pasteur"]], type = "charges")`.

## Simulation pour l’enseignement

`simulate_cfa()` génère des indicateurs continus à structure simple, avec facteurs de variance 1 et indicateurs de variance de population 1. La corrélation entre facteurs peut être une valeur commune ou une matrice admissible.

```r
sim <- simulate_cfa(
  n = 500,
  n_factors = 3,
  items_per_factor = 4,
  loading_range = c(0.50, 0.80),
  factor_correlation = 0.30,
  seed = 123
)

fit_sim <- cfa(sim$model, data = sim$data, std.lv = TRUE)
cfadiag(fit_sim)

sim$loadings         # Saturations de population.
sim$Phi              # Corrélations de population entre facteurs.
sim$population_model # Modèle générateur avec ses paramètres exacts.
```

Les estimations de l’échantillon fluctuent autour des paramètres de population. La simulation permet d’étudier ces fluctuations, la force des indicateurs ou une mauvaise spécification volontaire.

## Fonctions principales

| Fonction | Usage |
|---|---|
| `cfadiag()` | Produire le rapport global à partir d’un ajustement ou d’un `cfaModel` |
| `as.cfaModel()` | Convertir un ajustement `lavaan`, éventuellement par groupe |
| `cfaModel()` | Construire un conteneur de mesure à partir de matrices et de métadonnées |
| `compute_reliability()` | Calculer oméga, CR, alpha, H et leurs statuts |
| `compute_validity()` | Calculer AVE, MSV, ASV et le tableau de Fornell–Larcker |
| `compute_htmt()` | Calculer HTMT ou HTMT2 |
| `boot_htmt()` | Estimer les intervalles bootstrap de HTMT ou HTMT2 |
| `compute_fit_indices()` | Extraire les indices d’ajustement et leur version |
| `compute_diagnostics()` | Produire les signaux à examiner |
| `simulate_cfa()` | Générer des données et leurs paramètres de population |

Les fonctions `compute_reliability()`, `compute_validity()`, `compute_htmt()`, `compute_diagnostics()` et `boot_htmt()` attendent un objet `cfaModel`. Utilisez `as.cfaModel(fit)` pour les appeler séparément.

## Limites d’interprétation

- Les coefficients de fiabilité supposent une cible de score définie. Avec des saturations croisées ou des effets directs sur un indicateur, les indices cibles de fiabilité et l’AVE peuvent être suspendus pour les facteurs concernés. La colonne `statut` explique les valeurs manquantes; les facteurs non concernés peuvent conserver leurs indices.
- Un effet direct sur un indicateur n’est pas traité comme une simple saturation nulle : la version 0.2.3 lit les relations de mesure dans la table des paramètres de `lavaan`.
- Une covariance résiduelle est prise en compte dans les calculs compatibles; elle ne dispense pas de justifier la spécification.
- Le package ne calcule pas automatiquement une fiabilité hiérarchique du facteur général d’un modèle de second ordre ou bifactoriel.
- Avec un seul facteur, MSV, ASV et la distinction entre facteurs ne sont pas applicables. Certaines sorties restent absentes ou contiennent `NA`.
- Une solution non convergée, inadmissible ou une matrice singulière peut empêcher certains calculs. Lisez les signaux et les statuts avant de commenter les nombres.
- Un bon ajustement, une forte fiabilité ou des repères atteints ne suffisent pas à établir la validité d’un construit ou la pertinence d’un usage individuel des scores.

## Documentation et vignettes

```r
help(package = "cfadiag")
?cfadiag
?compute_reliability
?boot_htmt
```

Cinq vignettes accompagnent la version 0.2.3 :

1. [Prise en main de cfadiag](vignettes/cfadiag.Rmd)
2. [Comprendre les indices](vignettes/indices.Rmd)
3. [Données imparfaites](vignettes/donnees-imparfaites.Rmd)
4. [Au-delà de l’AFC simple](vignettes/au-dela-afc.Rmd)
5. [Simulation et enseignement](vignettes/simulation.Rmd)

Ces liens ouvrent les sources des vignettes dans le dépôt. Les versions compilées peuvent être consultées avec `browseVignettes("cfadiag")` lorsqu’elles sont incluses dans l’installation. Pour les construire lors d’une installation depuis GitHub, utilisez `build_vignettes = TRUE` avec `remotes::install_github()` après avoir installé `knitr` et `rmarkdown`.

## Développement et signalement de problèmes

Les tests se trouvent dans `tests/testthat`. Les vérifications de concordance avec `semTools` sont exécutées lorsque ce package est installé. Depuis la racine des sources :

```r
install.packages("testthat") # Si nécessaire.
testthat::test_local()
```

Pour signaler un problème dans l’onglet **Issues** du dépôt, fournissez un exemple reproductible, le message reçu, les versions de R, `lavaan` et `cfadiag`, et le résultat de `sessionInfo()`. Une petite simulation ou un jeu public est généralement suffisant pour reproduire un problème de calcul.

Les changements entre versions sont décrits dans [NEWS.md](NEWS.md).

## Auteur et licence

Développé et maintenu par **Daniel Coulombe**.

Le package est distribué sous licence **GNU General Public License, version 3 (GPL-3)**, conformément au champ `License` de `DESCRIPTION`.
