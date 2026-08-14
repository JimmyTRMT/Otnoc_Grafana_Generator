# Note technique - différences entre les OTNOC d'un site à l'autre

Le widget OTNOC est le même partout, mais **ce qu'il y a derrière change à chaque
site** : nomenclature des tags, codage des valeurs, numérotation. Cette note
recense ces différences, leur impact sur les blocs générés, et comment les
reconnaître quand quelque chose ne remonte pas.

---

## 1. Ce qui ne change jamais

Ces éléments sont identiques sur tous les sites et n'ont pas à être adaptés :

- le panel Grafana (type *Business HTML Graphics*) et ses 4 zones : Query A,
  HTML/SVG document, CSS, onRender ;
- la structure du tableau : début, fin, durée, libellé de la défaillance, n° OTNOC ;
- l'export CSV et son encodage (BOM UTF-8, séparateur `;`) ;
- l'accès à l'Historian : linked server `INSQL`, `OPENQUERY`, mode de récupération
  `Delta`, résolution 60 000 ms, `wwTimeZone = UTC` ;
- le principe de traitement : on récupère un point par changement de valeur, puis
  le JavaScript reconstitue des périodes de défaut.

Les blocs **HTML** et **CSS** sont donc rigoureusement les mêmes partout. Seuls
**Query A** et **onRender** varient, et uniquement à cause des points ci-dessous.

---

## 2. Axe de variation n°1 - la nomenclature des tags

C'est la différence la plus visible, et celle qui casse le SQL.

| Site | Exemple de tag | Points dans le nom |
|------|----------------|--------------------|
| Sète | `3BREF222_DEF_OTNOC1`, `3BREF100_DEF_OTNOC1` | non |
| Saint-Saulve | `OTNOC1_F2.DEF.PVFL`, `AU_MEM_F2.AU1.PVFL` | oui |

Saint-Saulve utilise une nomenclature de type DCS (`compound.block.paramètre`),
Sète une nomenclature KKS sans séparateur.

**Impact.** La requête historique interroge la table `WideHistory`, où **chaque
tag devient un nom de colonne**. Le provider `INSQL` sait le faire pour
`[3BREF222_DEF_OTNOC1]`, mais **pas** pour `[OTNOC1_F2.DEF.PVFL]` : il refuse la
requête à la préparation, avec ce message qui ne nomme jamais le tag fautif :

```
db query error: mssql: An error occurred while preparing the query "..."
for execution against OLE DB provider "INSQL" for linked server "INSQL".
```

**Solution.** Le fichier généré contient deux Query A :

| Bloc | Table interrogée | Les tags y sont... | À utiliser quand |
|------|------------------|--------------------|------------------|
| Query A | `WideHistory` | des **noms de colonnes** | tags sans point |
| VARIANTE de Query A | `History` + `PIVOT` | de simples **chaînes** dans un `IN (...)` | tags avec points |

La variante fait le pivot côté SQL Server, qui gère parfaitement les crochets
avec points. Le résultat envoyé au panel est identique dans les deux cas :
`DateTime` + une colonne par tag, nommée par le tag. Les autres blocs ne changent
pas.

---

## 3. Axe de variation n°2 - le codage de la valeur

Deuxième différence: la requête fonctionne, mais **le
tableau reste vide sans aucune erreur**.

| Codage dans l'automate | Site connu | Ce qu'affiche le tableau |
|------------------------|------------|--------------------------|
| booléen : 0 = pas de défaut, 1 = défaut actif | Sète | la période pendant laquelle la valeur est restée à 1 |
| incrémentation : compteur qui avance à chaque occurrence | Saint-Saulve | une ligne par série d'incréments |

**Solution.** Le fichier généré contient **deux blocs onRender**, un par codage :

| Bloc | À utiliser quand |
|------|------------------|
| onRender - VERSION 1/2 : POUR LE BOOLEEN | le tag vaut 0 ou 1 |
| onRender - VERSION 2/2 : POUR L'INCREMENTATION | le tag est un compteur |

Un seul des deux doit être collé. Ils ne diffèrent que par leur fonction de
détection : `calculateStateChanges` pour le booléen, `calculateIncrements` pour
le compteur. Le reste — dictionnaire des tags, tableau, export CSV — est
identique.

### Comment le mode `increment` est interprété

Un compteur ne donne pas d'état, seulement une progression. La règle appliquée :

- une occurrence **commence** quand le compteur se met à avancer ;
- elle **se termine** quand il cesse d'avancer ;
- un compteur qui n'avance qu'une fois (une impulsion par défaut) donne une ligne
  de durée `00:00:00` ;
- un compteur qui avance à chaque scrutation (compteur de temps de défaut) donne
  la durée réelle ;
- une **baisse** de valeur est traitée comme une remise à zéro, pas comme une
  occurrence.

Cette règle couvre les deux usages sans avoir à les distinguer à l'avance.

---

## 4. Axe de variation n°3 - la numérotation

| Point | Sète | Saint-Saulve |
|-------|------|--------------|
| Numéros contigus | non : 6 et 13 absents | oui, 1 à 25 |
| Ordre dans l'Excel | décroissant (25 → 1) | croissant |
| Numéros en double | non | **oui** : les 12 `AU_MEM_*` sont numérotés 1 à 12, comme les 12 premiers OTNOC |

Le n° OTNOC n'est **qu'un libellé affiché** : il ne sert pas de clé et n'a aucune
influence sur les requêtes. Deux familles de tags peuvent légitimement
partager la même numérotation. En revanche le **tag reste unique** et le contrôle
correspondant est maintenu, c'est lui qui sert de clé entre le SQL et le
JavaScript.

---

## 5. Axe de variation n°4 - le périmètre

Un site peut avoir plusieurs fours ou lignes, avec un panel par four :

- Sète : `3BREF222_*` et `3BREF100_*` ;
- Saint-Saulve : suffixes `_F1`, `_F2`, `_F3`.

**Un fichier Excel = un panel = un four.** Pour changer de four, on change les
tags dans l'onglet OTNOC, et on regénère. Vérifier que `panel_title` suit, sinon
on se retrouve à chercher pourquoi des données d'un four n'apparaissent pas dans
le panel d'un autre.

---

## 6. Axe de variation n°5 - l'historisation

Un tag peut exister dans l'automate, exister dans la table `Tag` de l'Historian,
et **n'avoir aucune valeur stockée**. Les trois situations se ressemblent vues du
panel — tableau vide — mais n'ont pas le même remède :

| Situation | Remède |
|-----------|--------|
| Nom de tag erroné dans l'Excel | corriger l'onglet OTNOC |
| Tag connu mais non historisé | activer l'historisation côté Historian |
| Tag historisé mais défaut jamais survenu | rien, le tableau vide est normal |

---

## 7. Symptôme → cause

| Ce qu'affiche Grafana | Cause | Où corriger |
|-----------------------|-------|-------------|
| `An error occurred while preparing the query ... INSQL` | tags avec points envoyés à `WideHistory` | utiliser la VARIANTE de Query A |
| `Error executing onRender` | le JavaScript a levé une exception (résultat sans colonne, bloc HTML absent) | vérifier que les 4 blocs sont collés ; le générateur pose désormais des gardes qui affichent un message lisible à la place |
| `Aucune série renvoyée par Query A` | la requête ne renvoie rien du tout | vérifier les noms de tags |
| `Query A ne renvoie aucune donnée sur la période` | tags sans valeur historisée sur la fenêtre | élargir la période, vérifier l'historisation |
| Tableau vide, **aucune erreur** | codage inadapté (compteur traité en booléen) | coller l'autre bloc onRender |
| Ligne « Tag Inconnu (...) » dans le tableau | un tag remonte du SQL mais est absent du dictionnaire onRender | recoller le bloc onRender après régénération |

**Query A et onRender doivent toujours venir de
la même génération.** Le SQL décide des tags remontés, le onRender contient leur
libellé, leur numéro et leur mode. Si on ne recolle que l'un des deux, le panel
n'affiche pas d'erreur — il affiche des lignes fausses ou rien.

---

## 8. Checklist pour un nouveau site

1. Récupérer la liste des tags **tels qu'ils sont nommés dans l'Historian**, pas
   dans l'automate ni dans la supervision.
2. Demander le codage de chaque famille de tags : booléen ou compteur.
3. Remplir l'Excel : onglet Config (`site_name`, `panel_title`), onglet OTNOC
   (tag, numéro, libellé).
4. Générer : `python generate_otnoc.py`.
5. Coller Query A. Si le message `preparing the query` apparaît, prendre la
   VARIANTE à la place.
6. Coller HTML, CSS, puis **un seul** des deux blocs onRender selon le codage.
7. Si le tableau reste vide sans erreur : essayer l'autre bloc onRender.
