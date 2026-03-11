# ⚽ Analyse Football International — Microsoft Fabric

![Power BI](https://img.shields.io/badge/Power%20BI-F2C811?style=for-the-badge&logo=powerbi&logoColor=black)
![Microsoft Fabric](https://img.shields.io/badge/Microsoft%20Fabric-0078D4?style=for-the-badge&logo=microsoft&logoColor=white)
![Apache Spark](https://img.shields.io/badge/Apache%20Spark-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)

## 📊 Présentation

Analyse historique des matchs de football international de **1875 à aujourd'hui**.  
Ce projet couvre plus de **47 000 matchs** et **130 000+ buts** sur toutes les compétitions internationales (FIFA et Hors FIFA).

Entièrement construit sur **Microsoft Fabric** avec une architecture Lakehouse en médaillon, une modélisation en schéma en étoile, des transformations Spark SQL et des rapports Power BI interactifs.

---

## 🛠️ Stack Technique

| Couche | Technologie |
|---|---|
| Stockage | Microsoft Fabric Lakehouse (Delta Lake) |
| Transformation | Apache Spark SQL / PySpark |
| Modélisation | Schéma en étoile |
| Reporting | Power BI (DAX) |
| Format des données | Tables Delta |

---

## 📐 Modèle de données

```
dim_date ───────────────────────────────┐
                                        │
dim_country ──► dim_match_country ──► matchs ◄── dim_tournament
                                        │
                                      goals ◄── dim_scorer
```

### Tables Gold

| Table | Description | Lignes |
|---|---|---|
| `matchs` | Données par match (scores, équipes, tournoi, winner, shootouts) | ~47 000 |
| `goals` | Table de faits — 1 ligne = 1 but (scorer, minute, penalty) | ~130 000 |
| `dim_country` | Pays avec noms historiques, confédération, canonical_name, flag_url | ~300 |
| `dim_match_country` | Table de pont match ↔ pays avec rôle (home/away) | ~94 000 |
| `dim_scorer` | Dimension joueurs | ~15 000 |
| `dim_tournament` | Dimension tournois | ~200 |
| `dim_date` | Dimension date (day, month, year, decade, date_key INT) | ~5 000 |

### Relations Power BI

| De | Vers | Direction |
|---|---|---|
| `dim_country[country_id]` | `dim_match_country[country_id]` | Both |
| `matchs[match_id]` | `dim_match_country[match_id]` | Both |
| `matchs[match_id]` | `goals[match_id]` | Both |
| `dim_tournament[tournament_id]` | `matchs[tournament_id]` | Both |
| `dim_date[date_key]` | `matchs[date_key]` | Both |
| `dim_scorer[scorer_id]` | `goals[scorer_id]` | Single |

---

## 📈 Pages du Dashboard

### 1. 🌍 Global Overview
Filtres : Confederation · Competition · Year

| Section | Tableaux |
|---|---|
| Apps | Top Apps (Team / Apps) |
| Wins & Losses | Top Win% · Top Loss% · Win Ratio · Loss Ratio |
| Goals | Top Scored (GF/Avg) · Top Conceded (GA/Avg) · Goals Ratio (GF/GA) |
| Penalties | PK Won (PK W%) · PK Lost (PK L%) |

### 2. 👤 Team Profile
Filtres : Team · Competition · Confederation · Decade · Year

KPIs : `Apps` · `W%` · `L%` · `D%` · `GF` · `GA` · `GD` · `GR` · `PKS` · `PKW%` · `PKL%` · `Avg GF` · `Avg GA`

Visuels : Drapeau dynamique · Team Form (10 derniers matchs) · Top Scorers

---

## 🧮 Mesures DAX

### Mesures de base

```dax
Matches Played = DISTINCTCOUNT(dim_match_country[match_id])

Total Goals =
VAR HOME = CALCULATE(SUM(matchs[home_score]), dim_match_country[role] = "home")
VAR AWAY = CALCULATE(SUM(matchs[away_score]), dim_match_country[role] = "away")
RETURN HOME + AWAY

Home Goals = CALCULATE(SUM(matchs[home_score]), dim_match_country[role] = "home", matchs[neutral] = FALSE)
Away Goals = CALCULATE(SUM(matchs[away_score]), dim_match_country[role] = "away", matchs[neutral] = FALSE)
Neutral Goals = [Total Goals] - ([Home Goals] + [Away Goals])

Goals Conceded =
VAR ConcededHome = CALCULATE(SUM(matchs[away_score]), dim_match_country[role] = "home")
VAR ConcededAway = CALCULATE(SUM(matchs[home_score]), dim_match_country[role] = "away")
RETURN ConcededHome + ConcededAway

Avg Goals Scored  = DIVIDE([Total Goals], [Matches Played], 0)
Avg Goals Conceded = DIVIDE([Goals Conceded], [Matches Played], 0)
Goals Ratio        = DIVIDE([Total Goals], [Goals Conceded], 0)
```

### Victoires / Défaites / Nuls

```dax
% Total Wins =
VAR WonNormal =
    CALCULATE(DISTINCTCOUNT(dim_match_country[match_id]), dim_match_country[role] = "home", matchs[home_score] > matchs[away_score], matchs[shootouts] = FALSE)
    + CALCULATE(DISTINCTCOUNT(dim_match_country[match_id]), dim_match_country[role] = "away", matchs[away_score] > matchs[home_score], matchs[shootouts] = FALSE)
RETURN DIVIDE(WonNormal, [Matches Played], 0)

% Total Losses =
VAR LostNormal =
    CALCULATE(DISTINCTCOUNT(dim_match_country[match_id]), dim_match_country[role] = "home", matchs[home_score] < matchs[away_score], matchs[shootouts] = FALSE)
    + CALCULATE(DISTINCTCOUNT(dim_match_country[match_id]), dim_match_country[role] = "away", matchs[away_score] < matchs[home_score], matchs[shootouts] = FALSE)
RETURN DIVIDE(LostNormal, [Matches Played], 0)

% Total Draws =
DIVIDE(CALCULATE(DISTINCTCOUNT(dim_match_country[match_id]), matchs[home_score] = matchs[away_score], matchs[shootouts] = FALSE), [Matches Played], 0)

Win Ratio =
VAR Wins = ... (home wins + away wins sans shootouts)
VAR NotWins = [Matches Played] - Wins
RETURN DIVIDE(Wins, NotWins, 0)

Loss Ratio =
VAR Losses = ... (home losses + away losses sans shootouts)
VAR NotLosses = [Matches Played] - Losses
RETURN DIVIDE(Losses, NotLosses, 0)
```

### Penalties

```dax
Total PK Sessions = CALCULATE(DISTINCTCOUNT(matchs[match_id]), matchs[shootouts] = TRUE)

% Matchs Gagnés avec Penalties =
VAR team = SELECTEDVALUE(dim_country[canonical_name])
VAR Won = CALCULATE(DISTINCTCOUNT(dim_match_country[match_id]), matchs[shootouts] = TRUE, matchs[winner] = team)
RETURN DIVIDE(Won, [Matches Played], 0)

% Matchs Perdus avec Penalties =
VAR team = SELECTEDVALUE(dim_country[canonical_name])
VAR Lost = CALCULATE(DISTINCTCOUNT(dim_match_country[match_id]), matchs[shootouts] = TRUE, matchs[winner] <> team, NOT ISBLANK(matchs[winner]))
RETURN DIVIDE(Lost, [Matches Played], 0)

PK Win Rate  = DIVIDE(Won PK sessions, Total PK Sessions, 0)
PK Loss Rate = DIVIDE(Lost PK sessions, Total PK Sessions, 0)
```

### Buteurs

```dax
Team Goals =
VAR team = SELECTEDVALUE(dim_country[canonical_name])
RETURN IF(ISBLANK(team),
    CALCULATE(COUNTROWS(goals), NOT ISBLANK(goals[scorer_id])),
    CALCULATE(COUNTROWS(goals), NOT ISBLANK(goals[scorer_id]), dim_scorer[scorer_team] = team)
)
```

### Tops & Flops (avec ALLSELECTED pour respecter les slicers)

```dax
Top Scorer Team     = CALCULATE(SELECTEDVALUE(dim_country[canonical_name]), TOPN(1, ALLSELECTED(dim_country[canonical_name]), [Total Goals], DESC))
Bottom Scorer Team  = CALCULATE(SELECTEDVALUE(dim_country[canonical_name]), TOPN(1, FILTER(ALLSELECTED(dim_country[canonical_name]), [Total Goals]>0), [Total Goals], ASC))
Top Team Win%       = CALCULATE(SELECTEDVALUE(dim_country[canonical_name]), TOPN(1, ALLSELECTED(dim_country[canonical_name]), [% Total Wins], DESC))
Top Team Loss%      = CALCULATE(SELECTEDVALUE(dim_country[canonical_name]), TOPN(1, ALLSELECTED(dim_country[canonical_name]), [% Total Losses], DESC))
Top Team PK Win Rate= CALCULATE(SELECTEDVALUE(dim_country[canonical_name]), TOPN(1, ALLSELECTED(dim_country[canonical_name]), [PK Win Rate], DESC))
Top Team PK Loss Rate=CALCULATE(SELECTEDVALUE(dim_country[canonical_name]), TOPN(1, ALLSELECTED(dim_country[canonical_name]), [PK Loss Rate], DESC))
Top Avg Goals Scored= CALCULATE(SELECTEDVALUE(dim_country[canonical_name]), TOPN(1, ALLSELECTED(dim_country[canonical_name]), [Avg Goals Scored], DESC))
Top Avg Goals Conceded=CALCULATE(SELECTEDVALUE(dim_country[canonical_name]), TOPN(1, ALLSELECTED(dim_country[canonical_name]), [Avg Goals Conceded], DESC))
```

### Forme de l'équipe

```dax
Team Form =
VAR team = SELECTEDVALUE(dim_country[canonical_name])
-- retourne 🟢 / 🔴 / ⚫ selon victoire/défaite/nul
-- trié par date_key dans le visuel matrice

Match Result By Year =
-- retourne 1 (plus de victoires), -1 (plus de défaites), 0 (équilibre)
-- utilisé pour la courbe de forme par année
```

### Labels HTML (visuel HTML Content)

```dax
Flag HTML = "<img src=" & SELECTEDVALUE(dim_country[flag_url]) & " width=160 height=100 /><br>" & SELECTEDVALUE(dim_country[canonical_name])

Win% Label  = "<div style='...'><span style='color:#00B04F'>🏆 " & FORMAT([% Total Wins],"0%") & " W</span></div>"
Loss% Label = "<div style='...'><span style='color:#FF4444'>❌ " & FORMAT([% Total Losses],"0%") & " L</span></div>"
Draw% Label = "<div style='...'><span style='color:#888888'>🤝 " & FORMAT([% Total Draws],"0%") & " D</span></div>"
-- + GF · GA · GD · GR · PKS · PKW · PKL · Avg GF · Avg GA labels
```

---

## 🧠 Défis Techniques Résolus

| Défi | Solution |
|---|---|
| Noms historiques des pays (URSS, Yougoslavie...) | Mapping `canonical_id` dans `dim_country` avec colonne `former_name` |
| Filtrage bidirectionnel home/away | Table de pont `dim_match_country` avec colonne `role` |
| Dates avant 1900 (bug Power BI) | `date_key` INT au format YYYYMMDD comme clé de relation |
| `neutral` et `shootouts` en texte | Conversion en BOOLEAN dans Spark SQL |
| TOPN non filtré par slicers | Remplacement de `ALL` par `ALLSELECTED` dans toutes les mesures Top/Bottom |
| Égalités dans TOPN | Critère de départage : `[Total Goals] * 1000000 + [Matches Played]` |

---

## 📥 Données

Les données brutes proviennent de Kaggle — téléchargement gratuit :

👉 [International Football Results 1872–2024 — Mart Jürisoo](https://www.kaggle.com/datasets/martj42/international-football-results-from-1872-to-2017)

Fichiers nécessaires : `results.csv` · `goalscorers.csv` · `shootouts.csv`· `former_names.csv`

Un fichier complémentaire `fifa_211_members_confederations.csv` a été généré pour enrichir
les données pays avec leur confédération FIFA (UEFA, CAF, CONMEBOL, etc.).
Ce fichier est inclus directement dans le repo dans le dossier `/data`.

Placer les fichiers dans le dossier `/data` avant d'exécuter les notebooks.

---

## 🚀 Comment Utiliser

### Prérequis
- Compte Microsoft Fabric (workspace actif)
- Power BI Desktop

### Étapes

```bash
git clone https://github.com/NacerEddine2911/international-football-analytics
```

1. Télécharger les données sur Kaggle et les placer dans `/data`
2. Importer les notebooks dans votre espace Fabric
3. Exécuter les notebooks dans l'ordre :
   - `NB_Bronze_to_Silver.ipynb`
   - `NB_Silver_to_Gold.ipynb`
4. Connecter Power BI Desktop au Lakehouse Gold
5. Ouvrir `RPT_Football_Histroqiue_Analysis.pbix`

---

## 🔧 Structure du Projet

```
📁 international-football-analytics
├── 📁 data/               ← placer ici les CSV Kaggle (non versionnés)
├── 📁 notebooks/
│   ├── NB_Bronze_to_Silver.ipynb
│   └── NB_Silver_to_Gold.ipynb
├── 📁 rapport/
│   └── RPT_Football_Histroqiue_Analysis.pbix
├── 📁 captures/
│   ├── global_overview.png
│   └── team_profile.png
└── README.md
```

---

## 📊 Glossaire des Métriques

| Métrique | Description |
|---|---|
| `Apps` | Apparitions (matchs joués) |
| `W%` | % Victoires hors penalties |
| `L%` | % Défaites hors penalties |
| `D%` | % Matchs nuls |
| `GF` | Buts marqués (Goals For) |
| `GA` | Buts encaissés (Goals Against) |
| `GD` | Différence de buts |
| `GR` | Ratio buts (GF/GA) |
| `PKS` | Sessions tirs au but |
| `PKW%` | % Sessions tirs au but gagnées |
| `PKL%` | % Sessions tirs au but perdues |

---

> **Stack** : Microsoft Fabric · Spark SQL · PySpark · DAX · Power BI · Delta Lake · Schéma en étoile
