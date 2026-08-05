# OTNOC Grafana Generator

Génère les 4 blocs à copier-coller dans un panel Grafana.
à partir d'un fichier Excel décrivant les OTNOC d'un site industriel.

## Contexte

Le widget OTNOC affiche en temps réel les défauts d'exploitation sur les usines de valorisation énergétique du groupe.

Chaque site a ses propres tags automate (nomenclature KKS différente), donc le
widget doit être régénéré pour chaque installation. Ce script automatise la
génération à partir d'un simple fichier Excel.

## Prérequis

- Python 3.8+
- `pip install openpyxl`

## Utilisation

1. Dupliquer ou changer `OTNOC_config.xlsx`
2. Remplir l'onglet **Config** (nom du site, titre du panel)
3. Remplir l'onglet **OTNOC** (une ligne par défaut)
4. Lancer :
```
   python generate_otnoc.py
```
5. Le script produit `OTNOC_<site>.txt` avec les 4 blocs à coller dans Grafana :
   - Query A (SQL Server)
   - HTML/SVG document
   - CSS
   - onRender

Le fichier contient en plus une **variante de Query A**, à n'utiliser que si la
requête principale échoue avec `error occurred while preparing the query ...
INSQL`. C'est le cas quand les tags du site contiennent des points
(ex. `AU_MEM_F1.AU1.PVFL`) : le provider INSQL ne sait pas les résoudre comme
colonnes de `WideHistory`. La variante lit la table `History` (les tags y sont
de simples chaînes) et fait le pivot côté SQL Server ; le résultat envoyé au
panel est identique, les autres blocs ne changent pas.

## Paramètres fixes

Modifiables en haut de `generate_otnoc.py` :

- `LINKED_SERVER = "INSQL"`
- `RESOLUTION_MS = 60000` (1 minute)
- `TABLE_HEIGHT_PX = 500`

## Structure attendue de l'Excel

**Onglet Config** :

| Paramètre    | Valeur          |
|--------------|-----------------|
| site_name    | Thiverval       |
| panel_title  | OTNOC 1         |

**Onglet OTNOC** :

| tag_name              | number | text                        |
|-----------------------|--------|-----------------------------|
| 3BREF100_DEF_OTNOC1   | 1      | Arrêt séquence alimenteur   |
| ...                   | ...    | ...                         |