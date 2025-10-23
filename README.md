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
