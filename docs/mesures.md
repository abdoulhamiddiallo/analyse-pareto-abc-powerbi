# Mesures DAX (70)

Toutes les mesures vivent dans la table `Pareto Metrics`, rangees par
dossier d'affichage.

## 01 Core

### Revenue

CA total sur le perimetre selectionne (axe Client neutralise). Denominateur de tous les %.

```dax
SUM ( Sheet1[Rev (mUSD)] )
```

Format : `#,##0.00`

### Revenue Total

Nombre de clients dans le perimetre selectionne.

```dax
CALCULATE ( [Revenue], ALL ( Sheet1 ) )
```

Format : `#,##0.00`

### Customers

Part du client dans le CA total.

```dax
CALCULATE ( DISTINCTCOUNT ( Sheet1[Client] ), ALL ( Sheet1 ) )
```

Format : `#,##0`

### Revenue %

CA moyen par client.

```dax
DIVIDE ( [Revenue], [Revenue Total] )
```

Format : `0.0%`

### Revenue per Customer

Rang du client par CA decroissant. Departage deterministe par nom en cas d'ex aequo.

```dax
DIVIDE ( [Revenue Total], [Customers] )
```

Format : `#,##0.00`

### Customer Count

CA de tout le portefeuille selectionne, tous filtres de Sheet1 neutralises.

```dax
DISTINCTCOUNT ( Sheet1[Client] )
```

Format : `#,##0`

### Revenue Total (Portfolio)

Part du CA total du portefeuille · reste correct dans une matrice groupee (ABC, groupe client).

```dax
CALCULATE ( [Revenue], ALL ( Sheet1 ) )
```

Format : `#,##0.00`

### Revenue % of Portfolio

Part du CA detenue par les N premiers clients, N venant de la table 'Top N Buckets'.

```dax
DIVIDE ( [Revenue], [Revenue Total] )
```

Format : `0.0%`

## 02 Pareto

### Customer Rank

CA cumule du rang 1 jusqu'au client courant.

```dax
VAR _CurRev = [Revenue]
VAR _CurCust = SELECTEDVALUE ( Sheet1[Client] )
VAR _Base =
    CALCULATETABLE (
        ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ),
        ALL ( Sheet1 )
    )
RETURN
    IF (
        NOT ISBLANK ( _CurRev ) && NOT ISBLANK ( _CurCust ),
        COUNTROWS (
            FILTER (
                _Base,
                [@Rev] > _CurRev || ( [@Rev] = _CurRev && Sheet1[Client] <= _CurCust )
            )
        )
    )
```

Format : `#,##0`

### Cumulative Revenue

Courbe de Pareto : part cumulee du CA.

```dax
VAR _CurRev = [Revenue]
VAR _CurCust = SELECTEDVALUE ( Sheet1[Client] )
VAR _Base =
    CALCULATETABLE (
        ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ),
        ALL ( Sheet1 )
    )
RETURN
    IF (
        NOT ISBLANK ( _CurRev ) && NOT ISBLANK ( _CurCust ),
        SUMX (
            FILTER (
                _Base,
                [@Rev] > _CurRev || ( [@Rev] = _CurRev && Sheet1[Client] <= _CurCust )
            ),
            [@Rev]
        )
    )
```

Format : `#,##0.00`

### Cumulative Revenue %

Part cumulee des clients (axe X normalise de la courbe de Lorenz).

```dax
DIVIDE ( [Cumulative Revenue], [Revenue Total] )
```

Format : `0.0%`

### Cumulative Customers %

Seuil pilote par le curseur. Sans selection : 80 % par defaut. Avec une plage : la borne haute.

```dax
DIVIDE ( [Customer Rank], [Customers] )
```

Format : `0.0%`

## 03 Cutoff & Segments

### Pareto Cutoff %

Nombre minimal de clients necessaires pour atteindre le seuil de CA cumule.

```dax
SELECTEDVALUE ( 'Pareto Cutoff'[Cutoff], 0.80 )
```

Format : `0 %`

### Vital Few Customers

CA genere par les clients du segment 'Vital few'.

```dax
VAR _Cut = [Pareto Cutoff %]
VAR _Total = [Revenue Total]
VAR _Base =
    CALCULATETABLE (
        ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ),
        ALL ( Sheet1 )
    )
VAR _WithPrior =
    ADDCOLUMNS (
        _Base,
        "@Prior",
        VAR _R = [@Rev]
        VAR _C = Sheet1[Client]
        RETURN
            SUMX (
                FILTER ( _Base, [@Rev] > _R || ( [@Rev] = _R && Sheet1[Client] < _C ) ),
                [@Rev]
            )
    )
RETURN
    COUNTROWS ( FILTER ( _WithPrior, DIVIDE ( [@Prior], _Total ) < _Cut ) ) + 0
```

Format : `#,##0`

### Vital Few Revenue

Part reelle du CA couverte par le segment 'Vital few'.

```dax
VAR _N = [Vital Few Customers]
VAR _Base =
    CALCULATETABLE (
        ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ),
        ALL ( Sheet1 )
    )
VAR _WithRank =
    ADDCOLUMNS (
        _Base,
        "@Rank",
        VAR _R = [@Rev]
        VAR _C = Sheet1[Client]
        RETURN
            COUNTROWS (
                FILTER ( _Base, [@Rev] > _R || ( [@Rev] = _R && Sheet1[Client] <= _C ) )
            )
    )
RETURN
    SUMX ( FILTER ( _WithRank, [@Rank] <= _N ), [@Rev] )
```

Format : `#,##0.00`

### Vital Few Revenue %

Part des clients que represente le segment 'Vital few'.

```dax
DIVIDE ( [Vital Few Revenue], [Revenue Total] )
```

Format : `0.0%`

### Vital Few Customers %

Helper 1/0 : le client courant appartient-il au segment 'Vital few'.

```dax
DIVIDE ( [Vital Few Customers], [Customers] )
```

Format : `0.0%`

### Is Vital Few

Libelle du segment : Vital few / Trivial many.

```dax
VAR _Rank = [Customer Rank]
RETURN
    IF ( NOT ISBLANK ( _Rank ) && _Rank <= [Vital Few Customers], 1, 0 )
```

Format : `0`

### Pareto Segment

Classification ABC standard sur le CA cumule : A <= 80%, B <= 95%, C au-dela.

```dax
IF ( ISBLANK ( [Customer Rank] ), BLANK (), IF ( [Is Vital Few] = 1, "Segment vital", "Queue longue" ) )
```

### ABC Class

Libelle long de la classe ABC pour les info-bulles.

```dax
VAR _Cum = [Cumulative Revenue %]
VAR _Prior = _Cum - [Revenue %]
RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( _Cum ), BLANK (),
        _Prior < 0.80, "A",
        _Prior < 0.95, "B",
        "C"
    )
```

### ABC Class Label

Part du CA detenue par le 1er client.

```dax
SWITCH (
    [ABC Class],
    "A", "A · Stratégiques (0-80 % du CA)",
    "B", "B · Intermédiaires (80-95 % du CA)",
    "C", "C · Queue longue (95-100 % du CA)",
    BLANK ()
)
```

## 04 Concentration

### Top 1 Share

Part du CA detenue par les 3 premiers clients.

```dax
VAR _Base =
    CALCULATETABLE ( ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ), ALL ( Sheet1 ) )
RETURN
    DIVIDE ( SUMX ( TOPN ( 1, _Base, [@Rev], DESC ), [@Rev] ), [Revenue Total] )
```

Format : `0.0%`

### Top 3 Share

Part du CA detenue par les 5 premiers clients.

```dax
VAR _Base =
    CALCULATETABLE ( ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ), ALL ( Sheet1 ) )
RETURN
    DIVIDE ( SUMX ( TOPN ( 3, _Base, [@Rev], DESC ), [@Rev] ), [Revenue Total] )
```

Format : `0.0%`

### Top 5 Share

Part du CA detenue par les 10 premiers clients.

```dax
VAR _Base =
    CALCULATETABLE ( ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ), ALL ( Sheet1 ) )
RETURN
    DIVIDE ( SUMX ( TOPN ( 5, _Base, [@Rev], DESC ), [@Rev] ), [Revenue Total] )
```

Format : `0.0%`

### Top 10 Share

Nombre de clients necessaires pour atteindre 80% du CA (regle de Pareto classique).

```dax
VAR _Base =
    CALCULATETABLE ( ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ), ALL ( Sheet1 ) )
RETURN
    DIVIDE ( SUMX ( TOPN ( 10, _Base, [@Rev], DESC ), [@Rev] ), [Revenue Total] )
```

Format : `0.0%`

### Customers to 80%

Part des clients necessaires pour atteindre 80% du CA.

```dax
VAR _Total = [Revenue Total]
VAR _Base =
    CALCULATETABLE ( ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ), ALL ( Sheet1 ) )
VAR _WithPrior =
    ADDCOLUMNS (
        _Base,
        "@Prior",
        VAR _R = [@Rev]
        VAR _C = Sheet1[Client]
        RETURN
            SUMX ( FILTER ( _Base, [@Rev] > _R || ( [@Rev] = _R && Sheet1[Client] < _C ) ), [@Rev] )
    )
RETURN
    COUNTROWS ( FILTER ( _WithPrior, DIVIDE ( [@Prior], _Total ) < 0.80 ) ) + 0
```

Format : `#,##0`

### Customers to 80% %

Ratio de Pareto observe, format '80 / xx%'.

```dax
DIVIDE ( [Customers to 80%], [Customers] )
```

Format : `0.0%`

### Pareto Ratio Label

Indice de Herfindahl-Hirschman (0-10 000). Mesure la concentration du portefeuille.

```dax
"80 / " & FORMAT ( [Customers to 80% %], "0 %" )
```

### HHI

Lecture du HHI selon les seuils antitrust usuels.

```dax
VAR _Total = [Revenue Total]
VAR _Base =
    CALCULATETABLE ( ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ), ALL ( Sheet1 ) )
RETURN
    SUMX ( _Base, DIVIDE ( [@Rev], _Total ) ^ 2 ) * 10000
```

Format : `#,##0`

### HHI Verdict

Coefficient de Gini du CA (0 = parfaitement reparti, 1 = totalement concentre).

```dax
VAR _H = [HHI]
RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( _H ), BLANK (),
        _H < 1500, "portefeuille diversifié",
        _H < 2500, "concentration modérée",
        "forte concentration"
    )
```

### Gini Coefficient

Diagnostic synthetique du risque de concentration client.

```dax
VAR _Total = [Revenue Total]
VAR _Base =
    CALCULATETABLE ( ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ), ALL ( Sheet1 ) )
VAR _N = COUNTROWS ( _Base )
VAR _WithRankAsc =
    ADDCOLUMNS (
        _Base,
        "@RankAsc",
        VAR _R = [@Rev]
        VAR _C = Sheet1[Client]
        RETURN
            COUNTROWS ( FILTER ( _Base, [@Rev] < _R || ( [@Rev] = _R && Sheet1[Client] <= _C ) ) )
    )
RETURN
    IF ( _N > 1, DIVIDE ( SUMX ( _WithRankAsc, ( 2 * [@RankAsc] - _N - 1 ) * [@Rev] ), _N * _Total ) )
```

Format : `0.000`

### Concentration Verdict

Couleur conditionnelle des barres : navy pour le segment vital, gris acier pour la queue longue.

```dax
VAR _G = [Gini Coefficient]
VAR _T3 = [Top 3 Share]
RETURN
    SWITCH (
        TRUE (),
        ISBLANK ( _G ), BLANK (),
        _T3 >= 0.50, "Risque élevé : les 3 premiers clients pèsent " & FORMAT ( _T3, "0 %" ) & " du CA",
        _G >= 0.45, "Risque modéré : distribution nettement inégale (Gini " & FORMAT ( _G, "0.00" ) & ")",
        "Risque maîtrisé : portefeuille relativement équilibré (Gini " & FORMAT ( _G, "0.00" ) & ")"
    )
```

### Top N Share

Legende de la carte Portefeuille.

```dax
VAR _N = SELECTEDVALUE ( 'Top N Buckets'[N] )
VAR _Base =
    CALCULATETABLE ( ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@Rev", [Revenue] ), ALL ( Sheet1 ) )
RETURN
    IF (
        NOT ISBLANK ( _N ),
        DIVIDE ( SUMX ( TOPN ( _N, _Base, [@Rev], DESC ), [@Rev] ), [Revenue Total] )
    )
```

Format : `0.0%`

## 05 Formatting

### Color - Bar

Couleur conditionnelle des barres selon la classe ABC.

```dax
IF ( [Is Vital Few] = 1, "#16324F", "#C9D4E2" )
```

### Color - Bar ABC

Couleur des etiquettes de donnees : blanc sur barre foncee, gris ardoise sinon.

```dax
SWITCH ( [ABC Class], "A", "#16324F", "B", "#4A87C7", "C", "#C9D4E2", "#E4E9F0" )
```

### Color - Data Label

Couleur d'accent du KPI de concentration : ambre si risque eleve, teal sinon.

```dax
IF ( [Is Vital Few] = 1, "#FFFFFF", "#5A6472" )
```

### Color - KPI Accent

Etiquette de barre sur deux lignes : valeur puis part du CA.

```dax
IF ( [Top 3 Share] >= 0.50, "#C77B22", "#1E7A6B" )
```

### Bar Label

Valeur a utiliser comme ligne constante sur l'axe secondaire (courbe cumulee).

```dax
IF (
    NOT ISBLANK ( [Revenue] ),
    FORMAT ( [Revenue], "0.00" ) & UNICHAR ( 10 ) & FORMAT ( [Revenue %], "0%" )
)
```

### Cutoff Line %

N'affiche la valeur cumulee que sur le client pivot, pour materialiser le point de bascule.

```dax
[Pareto Cutoff %]
```

Format : `0%`

### Cutoff Marker

Annotation affichee au point de bascule du Pareto.

```dax
IF ( [Customer Rank] = [Vital Few Customers], [Cumulative Revenue %] )
```

Format : `0.0%`

### Cutoff Callout

Titre dynamique du graphique de Pareto.

```dax
IF (
    [Customer Rank] = [Vital Few Customers],
    "Cutoff " & FORMAT ( [Pareto Cutoff %], "0%" ) & UNICHAR ( 10 ) & FORMAT ( [Cumulative Revenue], "0.00" ) & " mUSD"
)
```

## 06 Dynamic Titles

### Title - Pareto Chart

Sous-titre dynamique du graphique de Pareto.

```dax
"Analyse du chiffre d'affaires par client"
```

### Subtitle - Pareto Chart

Titre dynamique du tableau de detail.

```dax
"CA en mUSD  |  CA cumulé en %  ·  seuil piloté à " & FORMAT ( [Pareto Cutoff %], "0 %" ) & "  ·  ratio observé " & [Pareto Ratio Label]
```

### Title - Breakdown

Phrase choc pour la carte principale.

```dax
"Détail par client"
```

### KPI - Headline

Legende de la carte principale.

```dax
FORMAT ( [Vital Few Revenue %], "0%" ) & " du CA"
```

### KPI - Headline Sub

Synthese executive auto-redigee, a poser dans une carte multi-lignes.

```dax
"genere par " & FORMAT ( [Vital Few Customers], "0" ) & " clients (" & FORMAT ( [Vital Few Customers %], "0%" ) & " du portefeuille)"
```

### Narrative - Executive Summary

Texte d'info-bulle enrichi au survol d'un client.

```dax
VAR _Cut = [Pareto Cutoff %]
VAR _N = [Vital Few Customers]
VAR _NPct = [Vital Few Customers %]
VAR _RevPct = [Vital Few Revenue %]
VAR _Rev = [Vital Few Revenue]
VAR _Total = [Revenue Total]
VAR _Top3 = [Top 3 Share]
VAR _G = [Gini Coefficient]
RETURN
    IF (
        ISBLANK ( _Total ),
        BLANK (),
        "Sur " & FORMAT ( [Customers], "0" ) & " clients pesant " & FORMAT ( _Total, "#,##0.00" ) & " mUSD, "
            & FORMAT ( _N, "0" ) & " clients (" & FORMAT ( _NPct, "0%" ) & " du portefeuille) generent "
            & FORMAT ( _Rev, "#,##0.00" ) & " mUSD, soit " & FORMAT ( _RevPct, "0%" ) & " du chiffre d'affaires."
            & UNICHAR ( 10 ) & UNICHAR ( 10 )
            & "Ratio de Pareto observe : " & [Pareto Ratio Label] & " (la regle theorique serait 80 / 20)."
            & UNICHAR ( 10 )
            & "Les 3 premiers clients concentrent " & FORMAT ( _Top3, "0%" ) & " du CA · HHI " & FORMAT ( [HHI], "#,##0" ) & " (" & [HHI Verdict] & "), Gini " & FORMAT ( _G, "0.00" ) & "."
            & UNICHAR ( 10 ) & UNICHAR ( 10 )
            & [Concentration Verdict] & "."
    )
```

### Tooltip - Customer Story

Nombre de clients dans le contexte de filtre courant (utilisable par classe ABC).

```dax
VAR _C = SELECTEDVALUE ( Sheet1[Client] )
RETURN
    IF (
        ISBLANK ( _C ),
        BLANK (),
        _C & " · rang " & FORMAT ( [Customer Rank], "0" ) & " sur " & FORMAT ( [Customers], "0" )
            & UNICHAR ( 10 ) & FORMAT ( [Revenue], "#,##0.00" ) & " mUSD (" & FORMAT ( [Revenue %], "0.0%" ) & " du CA)"
            & UNICHAR ( 10 ) & "CA cumule a ce rang : " & FORMAT ( [Cumulative Revenue %], "0.0%" )
            & UNICHAR ( 10 ) & "Segment : " & [Pareto Segment] & " • Classe " & [ABC Class]
    )
```

### KPI - Portfolio Sub

Legende de la carte Ratio de Pareto.

```dax
FORMAT ( [Customers], "0" ) & " clients actifs  ·  " & FORMAT ( [Revenue per Customer], "#,##0.00" ) & " mUSD en moyenne"
```

### KPI - Ratio Sub

Legende de la carte Concentration.

```dax
"observé · la règle théorique serait 80 / 20"
```

### KPI - Concentration Sub

Version compacte de la synthese, calibree pour la carte 384 x 192 px.

```dax
"HHI  ·  Gini " & FORMAT ( [Gini Coefficient], "0.00" ) & "  ·  " & [HHI Verdict]
```

### Narrative - Panel

Serie de colonnes : CA des clients du segment vital (vide sinon).

```dax
VAR _Total = [Revenue Total]
RETURN
    IF (
        ISBLANK ( _Total ),
        BLANK (),
        "Sur " & FORMAT ( [Customers], "0" ) & " clients pesant " & FORMAT ( _Total, "#,##0.00" ) & " mUSD, "
            & FORMAT ( [Vital Few Customers], "0" ) & " clients (" & FORMAT ( [Vital Few Customers %], "0 %" ) & " du portefeuille) génèrent "
            & FORMAT ( [Vital Few Revenue], "#,##0.00" ) & " mUSD, soit " & FORMAT ( [Vital Few Revenue %], "0 %" ) & " du chiffre d'affaires."
            & UNICHAR ( 10 ) & UNICHAR ( 10 )
            & "Ratio observé " & [Pareto Ratio Label] & " contre 80 / 20 en théorie : la base est moins concentrée que le canon de Pareto."
            & UNICHAR ( 10 ) & UNICHAR ( 10 )
            & "Top 3 = " & FORMAT ( [Top 3 Share], "0 %" ) & " du CA. " & [Concentration Verdict] & "."
    )
```

## 07 Rapport FR

### CA segment vital

Serie de colonnes : CA des clients hors segment vital (vide sinon).

```dax
IF ( [Is Vital Few] = 1, [Revenue] )
```

Format : `0.00`

### CA queue longue

Courbe cumulee. Format vide : la courbe ne porte pas d'etiquette de donnees.

```dax
IF ( [Is Vital Few] = 0, [Revenue] )
```

Format : `0.00`

### CA cumulé %

Alias francais de la ligne de seuil, pour la legende du graphique.

```dax
[Cumulative Revenue %]
```

Format : `;;;`

### Seuil %

Alias francais du rang.

```dax
[Pareto Cutoff %]
```

Format : `0%`

### Rang

Alias francais du CA.

```dax
[Customer Rank]
```

Format : `0`

### CA mUSD

Alias francais de la part du CA.

```dax
[Revenue]
```

Format : `#,##0.00`

### % du CA

Alias francais du CA cumule.

```dax
[Revenue %]
```

Format : `0.0%`

### CA cumulé

Alias francais du CA cumule en %.

```dax
[Cumulative Revenue]
```

Format : `#,##0.00`

### % cumulé

Alias francais de la classe ABC.

```dax
[Cumulative Revenue %]
```

Format : `0.0%`

### Classe

Grand pourcentage de la phrase d'accroche.

```dax
[ABC Class]
```

### Headline %

Phrase d'accroche a droite du grand pourcentage.

```dax
FORMAT ( [Vital Few Revenue %], "0 %" )
```

### Headline phrase

Bandeau de synthese sur une seule ligne, en pied du panneau gauche.

```dax
"du CA est généré par " & FORMAT ( [Vital Few Customers], "0" ) & " clients sur " & FORMAT ( [Customers], "0" )
```

### Bandeau concentration

Barre de donnees textuelle. Uniquement sur les lignes client, jamais sur les sous-totaux.

```dax
VAR _Pivot =
    CALCULATE (
        MAXX (
            FILTER (
                ADDCOLUMNS ( VALUES ( Sheet1[Client] ), "@R", [Customer Rank] ),
                [@R] = [Vital Few Customers]
            ),
            Sheet1[Client]
        ),
        ALL ( Sheet1 )
    )
RETURN
    "Ratio " & [Pareto Ratio Label]
        & "     ·     Top 3 = " & FORMAT ( [Top 3 Share], "0 %" )
        & "     ·     HHI " & FORMAT ( [HHI], "#,##0" )
        & "     ·     Gini " & FORMAT ( [Gini Coefficient], "0.00" )
        & "     ·     bascule chez " & _Pivot
```

### Barre

Marqueur unique au client pivot. Son etiquette sert d'annotation de bascule sur la courbe.

```dax
VAR _Max = CALCULATE ( MAX ( Sheet1[Rev (mUSD)] ), ALL ( Sheet1 ) )
VAR _N = ROUND ( DIVIDE ( [Revenue], _Max ) * 8, 0 )
RETURN
    IF (
        ISINSCOPE ( Sheet1[Client] ) && NOT ISBLANK ( [Revenue] ),
        REPT ( UNICHAR ( 9608 ), MAX ( _N, 1 ) )
    )
```

### Point de bascule

Nom du client survole.

```dax
IF ( [Customer Rank] = [Vital Few Customers], [Cumulative Revenue %] )
```

Format : `0 %`

## 08 Info-bulle

### Tooltip - Client

Ligne de position du client dans le classement.

```dax
SELECTEDVALUE ( Sheet1[Client], "Tous les clients" )
```

### Tooltip - Position

Phrase de lecture au survol.

```dax
VAR _R = [Customer Rank]
RETURN
    IF (
        ISBLANK ( _R ),
        BLANK (),
        "Rang " & FORMAT ( _R, "0" ) & " sur " & FORMAT ( [Customers], "0" )
            & "   ·   Classe " & [ABC Class]
            & "   ·   " & [Pareto Segment]
    )
```

### Tooltip - Verdict

```dax
VAR _R = [Customer Rank]
VAR _N = [Vital Few Customers]
RETURN
    IF (
        ISBLANK ( _R ),
        BLANK (),
        IF (
            _R <= _N,
            "Fait partie des " & FORMAT ( _N, "0" ) & " clients qui génèrent "
                & FORMAT ( [Vital Few Revenue %], "0 %" ) & " du chiffre d'affaires.",
            "Au-delà du seuil de " & FORMAT ( [Pareto Cutoff %], "0 %" ) & " : "
                & FORMAT ( _R - _N, "0" ) & " rang(s) après le point de bascule."
        )
    )
```
