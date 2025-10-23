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

