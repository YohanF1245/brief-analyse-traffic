# BRIEF ANALYSE TRAFFIC
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
![image](/mld/diagram211025.png)

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
1. Accidents par année et par gravité
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
2. Accidents par tranche d'âge
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
3. Accidents par type de véhicules
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
4. Gravité des accidents par type de véhicules
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