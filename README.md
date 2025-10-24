# BRIEF ANALYSE TRAFFIC
## Sommaire
- [BRIEF ANALYSE TRAFFIC](#brief-analyse-traffic)
  - [Sommaire](#sommaire)
  - [Gestion de projet](#gestion-de-projet)
  - [Technos utilisées :](#technos-utilisées-)
  - [Conception mcd](#conception-mcd)
  - [Récuperation des données de l'api](#récuperation-des-données-de-lapi)
  - [Création de la base de données :](#création-de-la-base-de-données-)
  - [Requêtes analytiques](#requêtes-analytiques)
    - [1. Accidents par année et par gravité](#1-accidents-par-année-et-par-gravité)
    - [2. Accidents par tranche d'âge](#2-accidents-par-tranche-dâge)
    - [3. Accidents par type de véhicules](#3-accidents-par-type-de-véhicules)
    - [4. Gravité des accidents par type de véhicules](#4-gravité-des-accidents-par-type-de-véhicules)
    - [5. Accidents par heure de la journée](#5-accidents-par-heure-de-la-journée)
    - [6. Place du mort](#6-place-du-mort)
    - [7. Port de la ceinture](#7-port-de-la-ceinture)
    - [8. Accidents selon la manoeuvre](#8-Accidents-selon-la-manoeuvre)
    - [9. Accidents selon le genre](#9-Accidents-selon-le-genre)
    - [10. Heatmap meteo lumiere](#7-Heatmap-meteo-lumiere)
## Gestion de projet
Travail collaboratif en pull requests avec deux reviewers avant merge.
## Technos utilisées : 
- Git
- Jupyter Notebook
- Sqlite3
- Pandas
- Numpy
- Matplotlib
## Conception mcd
Nous avons opté pour un schéma en étoile centré sur le fait "accident" avec les dimensions :
- usager
- temps 
- vehicule
- contexte
- contexte
![image](/mld/diagram.png)

## Récuperation des données de l'api
L'api à une limitation sur le volume de données récupérable, il fallait diviser la rêquete et fetch les données mois par mois, pour rester sous le seuil de 10000 entrées récupérable.
```
utilisation avec les parametres mois et année
https://public.opendatasoft.com/api/explore/v2.1/catalog/datasets/accidents-corporels-de-la-circulation-millesime/records?where=an%3D%27""ANNEE""%27%20and%20mois%3D%27""03""%27&limit=10&offset=0&timezone=UTC&include_links=false&include_app_metas=false

```
À noter que dans une optique d'automatisation d'etl, on peut imaginer que le job va tourner tout les mois, et recuperer les données du mois precedant (via les parametres année / mois qui permettent de cibles quelle plage de donées récuperer).

## Création de la base de données : 

``` Python
import sqlite3

conn = sqlite3.connect('accidents.db')
cursor = conn.cursor()

cursor.execute('''
CREATE TABLE IF NOT EXISTS dim_localisation (
  id_localisation TEXT PRIMARY KEY,
  nom_commune TEXT,
  agglomeration TEXT,
  departement_code TEXT,
  nom_departement TEXT,
  commune_code TEXT,
  adresse_postale TEXT,
  code_postal TEXT
);
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS dim_contexte (
  id_contexte TEXT PRIMARY KEY,
  lumiere TEXT,
  intersection TEXT,
  condition_atmos TEXT,
  collision TEXT,
  type_surface TEXT,
  regime_circulation TEXT,
  voie_reservee TEXT,
  proximite_ecole TEXT,
  nombre_voie_circulation INTEGER,
  categorie_route TEXT,
  pente TEXT,
  infrastructure TEXT,
  situation_accident TEXT
);
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS dim_temps (
  id_date TEXT PRIMARY KEY,
  annee INTEGER,
  mois INTEGER,
  jour INTEGER,
  hrmn TEXT,
  date TEXT
);
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS fact_accidents (
  id_accident TEXT PRIMARY KEY,
  num_acc INTEGER,
  obstacle_mobile_heurte TEXT,
  obstacle_fixe_heurte TEXT,
  id_localisation TEXT,
  id_contexte TEXT,
  id_date TEXT,
  FOREIGN KEY (id_localisation) REFERENCES dim_localisation(id_localisation),
  FOREIGN KEY (id_contexte) REFERENCES dim_contexte(id_contexte),
  FOREIGN KEY (id_date) REFERENCES dim_temps(id_date)
);
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS dim_usager (
  id_usager TEXT PRIMARY KEY,
  annee_naissance INTEGER,
  sexe TEXT,
  gravite_accident TEXT,
  securite TEXT,
  locp TEXT,
  place TEXT,
  categorie_usager TEXT,
  booster TEXT,
  trajet TEXT,
  action_pieton TEXT,
  id_accident TEXT,
  FOREIGN KEY(id_accident) REFERENCES fact_accidents(id_accident)
);
''')

cursor.execute('''
CREATE TABLE IF NOT EXISTS dim_vehicule (
  id_vehicule TEXT PRIMARY KEY,
  catv TEXT,
  num_veh TEXT,
  choc TEXT,
  manoeuvre_av_accident TEXT,
  id_accident TEXT,
  FOREIGN KEY(id_accident) REFERENCES fact_accidents(id_accident)
);
''')

conn.commit()
conn.close()
```

## Requêtes analytiques
### 1. Accidents par année et par gravité
```python
accidents_par_annee = pd.read_sql_query('''
    SELECT count(*) AS nombre_accidents, dt.annee
    FROM dim_temps dt
    GROUP BY dt.annee
    ''',conn)
gravite_par_annee = pd.read_sql_query('''
     SELECT 
        dt.annee,
        du.gravite_accident,
    COUNT(*) AS nombre_accidents
    FROM dim_usager AS du
    INNER JOIN fact_accidents AS fa
    ON fa.id_accident = du.id_accident
    INNER JOIN dim_temps AS dt
    ON dt.id_date = fa.id_date
    GROUP BY dt.annee, du.gravite_accident  ''',
    conn) 

```
![image](/accidents-par-annees.png)
### 2. Accidents par tranche d'âge
``` python
query = """
SELECT
    COUNT(*) AS total,
    (annee_naissance / 10) * 10 AS decennie,
    gravite_accident
FROM dim_usager
LEFT JOIN fact_accidents
    ON fact_accidents.id_accident = dim_usager.id_accident
WHERE categorie_usager = 'Conducteur'
GROUP BY decennie, gravite_accident
ORDER BY decennie;
"""
```
![image](/accidents-par-tranche-age.png)
![image](/evolution-accidents-par-tranche-age.png)
### 3. Accidents par type de véhicules
``` python
query = """
SELECT
    CASE
        WHEN catv LIKE '%VL%' THEN 'Voiture particulière'
        WHEN catv LIKE 'VU%' OR catv LIKE 'PL%' OR catv LIKE 'Tracteur routier%' THEN 'Utilitaire / Poids lourd'
        WHEN catv LIKE 'Motocyclette%' OR catv LIKE 'Scooter >%' THEN '2-roues motorisés'
        WHEN catv LIKE 'Cyclomoteur%' OR catv LIKE 'Scooter <%' OR catv LIKE 'Quad%' THEN 'Cyclomoteurs et quads'
        WHEN catv LIKE 'Bicyclette%' THEN 'Vélo'
        WHEN catv LIKE 'Autocar%' OR catv LIKE 'Autobus%' THEN 'Transport collectif'
        WHEN catv LIKE 'Tramway%' OR catv LIKE 'Train%' THEN 'Ferroviaire'
        WHEN catv LIKE 'Tracteur agricole%' OR catv LIKE 'Engin spécial%' THEN 'Engin / Agricole'
        WHEN catv LIKE 'Voiturette%' THEN 'Voiturette'
        ELSE 'Autre'
    END AS type_vehicule,
    COUNT(*) AS total
FROM dim_vehicule
GROUP BY type_vehicule
ORDER BY total DESC;
"""
```
![image](/accidents-par-types-vehicules.png)
### 4. Gravité des accidents par type de véhicules
``` Python
SQL = """
SELECT
    CASE
        WHEN dv.catv LIKE '%VL%' THEN 'Voiture particulière'
        WHEN dv.catv LIKE 'VU%' OR dv.catv LIKE 'PL%' OR dv.catv LIKE 'Tracteur routier%' THEN 'Utilitaire / Poids lourd'
        WHEN dv.catv LIKE 'Motocyclette%' OR dv.catv LIKE 'Scooter >%' THEN '2-roues motorisés'
        WHEN dv.catv LIKE 'Cyclomoteur%' OR dv.catv LIKE 'Scooter <%' OR dv.catv LIKE 'Quad%' THEN 'Cyclomoteurs et quads'
        WHEN dv.catv LIKE 'Bicyclette%' THEN 'Vélo'
        WHEN dv.catv LIKE 'Autocar%' OR dv.catv LIKE 'Autobus%' THEN 'Transport collectif'
        WHEN dv.catv LIKE 'Tramway%' OR dv.catv LIKE 'Train%' THEN 'Ferroviaire'
        WHEN dv.catv LIKE 'Tracteur agricole%' OR dv.catv LIKE 'Engin spécial%' THEN 'Engin / Agricole'
        WHEN dv.catv LIKE 'Voiturette%' THEN 'Voiturette'
        ELSE 'Autre'
    END AS type_vehicule,
    COALESCE(du.gravite_accident, 'Non renseigné') AS gravite_accident,
    COUNT(*) AS total
FROM fact_accidents AS fa
JOIN dim_usager AS du
    ON fa.id_accident = du.id_accident
JOIN dim_vehicule AS dv
    ON fa.id_accident = dv.id_accident
WHERE du.categorie_usager = 'Conducteur'
GROUP BY type_vehicule, du.gravite_accident
ORDER BY type_vehicule, du.gravite_accident;
"""
```
![image](/gravite-accidents-voitures-particulieres.png)
![image](/gravite-accident-type-vehicule.png)

### 5. Accidents par heure de la journée

``` Python
SQL = """
SELECT
    -- Extraction de l'heure en nombre entier
    CAST(
        CASE
            WHEN hrmn ~ '^[0-9]{4}$' THEN SUBSTRING(hrmn FROM 1 FOR 2)
            WHEN hrmn ~ '^[0-9]{1,2}:[0-9]{2}$' THEN SPLIT_PART(hrmn, ':', 1)
            WHEN hrmn ~ '^[0-9]{1,2}h[0-9]{0,2}$' THEN SPLIT_PART(hrmn, 'h', 1)
            ELSE hrmn
        END AS INTEGER
    ) AS heure,

    COUNT(*) AS nb_accidents
FROM dim_temps
GROUP BY heure
ORDER BY heure;

"""
```
![image](/accidentsheure.png)

### 6. Place du mort

``` Python
SQL = """
-- Taux de mortalité par place dans le véhicule
WITH
-- Comptage total des usagers par place
total_usagers AS (
    SELECT
        place,
        COUNT(*) AS total
    FROM dim_usager
    GROUP BY place
),

-- Comptage des usagers tués par place
usagers_tues AS (
    SELECT
        place,
        COUNT(*) AS nb_tues
    FROM dim_usager
    WHERE LOWER(gravite_accident) LIKE '%tué%'
    GROUP BY place
),

-- Fusion + calcul du taux de mortalité
taux_mortalite AS (
    SELECT
        t.place,
        COALESCE(t.total, 0) AS total,
        COALESCE(u.nb_tues, 0) AS nb_tues,
        ROUND(100.0 * COALESCE(u.nb_tues, 0) / COALESCE(t.total, 1), 2) AS taux_mortalite
    FROM total_usagers t
    LEFT JOIN usagers_tues u ON t.place = u.place
)

-- Sélection finale avec correspondance des libellés
SELECT
    CASE CAST(place AS INTEGER)
        WHEN 1 THEN 'Conducteur'
        WHEN 2 THEN 'Passager avant'
        WHEN 3 THEN 'Arrière gauche'
        WHEN 4 THEN 'Arrière milieu'
        WHEN 5 THEN 'Arrière droit'
        WHEN 6 THEN 'Autre rang'
        WHEN 7 THEN 'Extérieur'
        WHEN 8 THEN 'Inconnu'
        WHEN 9 THEN 'Autre'
        ELSE 'Non défini'
    END AS place_label,
    total,
    nb_tues,
    taux_mortalite
FROM taux_mortalite
ORDER BY taux_mortalite DESC;


"""
```
![image](/mortaliteplace.png)

### 7. Port de la ceinture

``` Python
SQL = """
WITH securite_simplifiee AS (
    SELECT
        id_usager,
        id_accident,
        -- Simplification de la colonne "securite"
        CASE
            WHEN securite IS NULL THEN 'Inconnu'
            WHEN LOWER(securite) LIKE '%ceinture%' 
              OR LOWER(securite) LIKE '%casque%' 
              OR LOWER(securite) LIKE '%retenue%' 
              OR LOWER(securite) LIKE '%airbag%' 
            THEN 'Porté'
            WHEN LOWER(securite) LIKE '%non%' 
              OR LOWER(securite) LIKE '%aucun%' 
              OR LOWER(securite) LIKE '%abs%' 
              OR LOWER(securite) LIKE '%sans%' 
            THEN 'Non porté'
            ELSE 'Inconnu'
        END AS securite_simple,

        -- Simplification de la gravité
        CASE
            WHEN gravite_accident IS NULL THEN 'Inconnu'
            WHEN LOWER(gravite_accident) LIKE '%tué%' THEN 'Tué'
            WHEN LOWER(gravite_accident) LIKE '%bless%' THEN 'Blessé'
            WHEN LOWER(gravite_accident) LIKE '%indemne%' THEN 'Indemne'
            ELSE 'Autre'
        END AS gravite_simple
    FROM dim_usager
),

-- Comptage du nombre d’usagers par sécurité et gravité
compte AS (
    SELECT
        securite_simple,
        gravite_simple,
        COUNT(*) AS nb_usagers
    FROM securite_simplifiee
    GROUP BY securite_simple, gravite_simple
),

-- Calcul du pourcentage dans chaque catégorie de sécurité
totaux AS (
    SELECT
        securite_simple,
        SUM(nb_usagers) AS total_usagers
    FROM compte
    GROUP BY securite_simple
)

SELECT
    c.securite_simple,
    c.gravite_simple,
    c.nb_usagers,
    ROUND((c.nb_usagers * 100.0 / t.total_usagers), 2) AS pct
FROM compte c
JOIN totaux t
  ON c.securite_simple = t.securite_simple
ORDER BY c.securite_simple, c.gravite_simple;

"""
```
![image](/portceinture.png)

### 8. Accidents selon la manoeuvre

``` Python
SQL = """
SELECT 
    v.manoeuvre_av_accident AS manoeuvre,
    COUNT(DISTINCT v.id_accident) AS nb_accidents
FROM dim_vehicule v
WHERE v.manoeuvre_av_accident IS NOT NULL
  AND v.manoeuvre_av_accident NOT IN ('Sans changement de direction', '26','25','-1')
GROUP BY v.manoeuvre_av_accident
ORDER BY nb_accidents DESC;

"""
```
![image](/accident_selon_la_manoeuvre.png)

### 9. Accidents selon le genre

``` Python
SQL = """
SELECT 
    u.sexe,
    COUNT(DISTINCT u.id_accident) AS nb_accidents
FROM dim_usager u
WHERE u.sexe IS NOT NULL
GROUP BY u.sexe
ORDER BY nb_accidents DESC;

"""
```
![image](/accident_femme_homme.png)

### 10. Heatmap meteo lumiere

``` Python
SQL = """
SELECT 
    c.lumiere AS condition_lumiere,
    c.condition_atmos AS condition_meteo,
    COUNT(*) AS nb_accidents
FROM fact_accidents f
LEFT JOIN dim_contexte c ON c.id_contexte = f.id_contexte
WHERE c.lumiere IS NOT NULL
  AND c.condition_atmos IS NOT NULL
  AND c.lumiere <> 'Plein jour'
  AND c.condition_atmos <> 'Normal'
GROUP BY c.lumiere, c.condition_atmos;

"""
```
![image](/heatmap_lumiere_et_meteo.png)


