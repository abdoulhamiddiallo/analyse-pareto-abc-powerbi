# Modele de donnees

Quatre tables, aucune relation. Les trois tables auxiliaires sont
deconnectees a dessein : elles alimentent des segments et des seuils que
les mesures lisent par `SELECTEDVALUE`, ce qui est le motif classique
d'une analyse de Pareto parametrable.

## Pareto Cutoff

Parametre what-if : seuil de concentration Pareto (50% a 100%, pas de 1%).

Colonnes : `Cutoff`, `Ordre`, `Seuil`

## Pareto Metrics

Table hote des mesures Pareto.

Colonnes : `_`

70 mesures, detaillees dans [mesures.md](mesures.md).

## Sheet1

Colonnes : `Client`, `Rev (mUSD)`, `_2 Customer Group`, `_3 Customer Group`, `ABC Class (static)`, `Rank (static)`, `Ordre`, `Client et part`

## Top N Buckets

Table deconnectee : paliers de concentration Top-N.

Colonnes : `Bucket`, `N`, `Sort`

## Parametre de source

Le parametre Power Query `DataFolder` porte le chemin absolu du dossier
`data`. Il est a renseigner une fois apres le clone, rien d'autre
n'est a modifier.
