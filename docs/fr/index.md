---
machine_translated: true
machine_translated_rev: 72537a2d527c63c07aa5d2361a8829f3895cf2bd
toc_priority: 0
toc_title: "Aper\xE7u"
---

# Qu'est-ce que ClickHouse? {#what-is-clickhouse}

ClickHouse est un système de gestion de base de données orienté colonne (SGBD) pour le traitement analytique en ligne des requêtes (OLAP).

Dans un SGBD “normal” orienté ligne, les données sont stockées de cette façon :

| Rangée | WatchID     | JavaEnable | Intitulé                         | GoodEvent | EventTime           |
|--------|-------------|------------|----------------------------------|-----------|---------------------|
| \#0    | 89354350662 | 1          | Relations Avec Les Investisseurs | 1         | 2016-05-18 05:19:20 |
| \#1    | 90329509958 | 0          | Contacter                        | 1         | 2016-05-18 08:10:20 |
| \#2    | 89953706054 | 1          | Mission                          | 1         | 2016-05-18 07:38:00 |
| \#N    | …           | …          | …                                | …         | …                   |

En d'autres termes, toutes les valeurs liées à une ligne sont physiquement stockées les unes à côté des autres.

Exemples de SGBD orienté ligne : MySQL, Postgres et MS SQL Server.

Dans un SGBD orienté colonne, les données sont stockées comme ceci :

| Rangée:     | \#0                              | \#1                 | \#2                 | \#N |
|-------------|----------------------------------|---------------------|---------------------|-----|
| WatchID:    | 89354350662                      | 90329509958         | 89953706054         | …   |
| JavaEnable: | 1                                | 0                   | 1                   | …   |
| Intitulé:   | Relations Avec Les Investisseurs | Contacter           | Mission             | …   |
| GoodEvent:  | 1                                | 1                   | 1                   | …   |
| EventTime:  | 2016-05-18 05:19:20              | 2016-05-18 08:10:20 | 2016-05-18 07:38:00 | …   |

Ces exemples montrent dans quel ordre sont organisées les données. Les valeurs des différentes colonnes sont stockées séparément, et les données d'une même colonne sont stockées ensemble.

Exemples de SGBD orienté colonne: Vertica, Paraccel (matrice Actian et Amazon Redshift), Sybase IQ, Exasol, Infobright, InfiniDB, MonetDB (VectorWise et Actian Vector), LucidDB, SAP HANA, Google Dremel, Google PowerDrill, Druid et kdb+.

Le façon dont sont stockées les données est adaptée à différents scénarios. Un scénario d'accès au données fait référence à ce dont les requêtes sont constituées, et dans quelle proportion; combien de données sont lues pour chaque type de requête - lignes, colonnes, et octets; les relations entre la lecture et la mise à jour des données; la taille de la données et comment elle est localement utilisée; si des transactions sont effectuées, et leur niveau d'isolation ; les pré-requis en termes de réplication de données et d'intégrité logique; les besoins en latence et en débit pour chaque type de requête, etc...

Plus la charge sur le système est élevée, plus il est important de personnaliser le système configuré pour correspondre aux exigences du scénario d'utilisation, et plus cette personnalisation devient fine. Il n'y a pas de système qui soit adapté à plusieurs scénarios significativement différents. Si un système est adaptable à un large ensemble de scénarios, avec une charge élevée, le système traitera tous les scénarios de manière également médiocre, ou fonctionnera bien pour un ou seulement quelques-uns des scénarios possibles.

## Propriétés clés du scénario OLAP {#key-properties-of-olap-scenario}

-   La grande majorité des requêtes concernent l'accès en lecture.
-   Les données sont mises à jour en lots assez importants (\> 1000 lignes), et non pas par des lignes seules; ou elles ne sont pas mises à jour du tout.
-   Les données sont ajoutées à la base de données, sans modification de l'existant.
-   Pour les lectures, un assez grand nombre de lignes sont extraites de la base de données, mais seulement un petit sous-ensemble de colonnes.
-   Les Tables sont “wide,” ce qui signifie qu'elles contiennent un grand nombre de colonnes.
-   Les requêtes sont relativement peu soutenues (généralement des centaines de requêtes par serveur ou moins par seconde).
-   Pour les requêtes simples, les latences autour de 50 ms sont acceptées.
-   Les valeurs de colonne sont de taille assez petite: nombres et chaînes courtes (par exemple, 60 octets par URL).
-   Nécessite un débit élevé lors du traitement d'une seule requête (jusqu'à des milliards de lignes par seconde par serveur).
-   Le mode transactionnel n'est pas nécessaire.
-   Faibles exigences en matière de cohérence des données.
-   Il y a une grande table par requête. Toutes les autres tables sont petites.
-   Un résultat de requête est significativement plus petit que les données source. En d'autres termes, les données sont filtrées ou agrégées, de sorte que le résultat s'intègre dans la RAM d'un seul serveur.

Il est facile de voir que le scénario OLAP est très différent des autres scénarios populaires (tels que OLTP ou key-Value access). Il n'est donc pas logique d'essayer d'utiliser OLTP ou une base de données clé-valeur pour traiter les requêtes analytiques si vous voulez obtenir des performances décentes. Par exemple, si vous essayez d'utiliser MongoDB ou Redis pour l'analyse, vous obtiendrez des performances très médiocres par rapport aux bases de données OLAP.

## Pourquoi les bases de données orientées colonne fonctionnent mieux dans le scénario OLAP {#why-column-oriented-databases-work-better-in-the-olap-scenario}

Les bases de données orientées colonne sont mieux adaptées aux scénarios OLAP: elles sont au moins 100 fois plus rapides dans le traitement de la plupart des requêtes. Les raisons sont expliquées en détail ci-dessous, mais le principe est plus facile à démontrer visuellement:

**SGBD orienté ligne**

![Row-oriented](images/row-oriented.gif#)

**SGBD orienté colonne**

![Column-oriented](images/column-oriented.gif#)

Vous voyez la différence ?

### Entrée/sortie {#inputoutput}

1.  Pour une requête analytique, seul un petit nombre de colonnes de la table doit être lu. Dans une base de données orientée colonne, vous pouvez lire uniquement les données dont vous avez besoin. Par exemple, si vous avez besoin de 5 colonnes sur 100, Vous pouvez vous attendre à une réduction de 20 fois des e / s.
2.  Puisque les données sont lues en paquets, il est plus facile de les compresser. Les données dans les colonnes sont également plus faciles à compresser. Cela réduit d'autant le volume d'E/S.
3.  En raison de la réduction des E/S, plus de données s'insèrent dans le cache du système.

Par exemple, la requête “count the number of records for each advertising platform” nécessite la lecture de la colonne “advertising platform ID”, qui prend 1 octet non compressé. Si la majeure partie du trafic ne provient pas de plates-formes publicitaires, vous pouvez vous attendre à une compression d'au moins 10 fois de cette colonne. Lors de l'utilisation d'un algorithme de compression rapide, la décompression des données est possible à une vitesse d'au moins plusieurs gigaoctets de données non compressées par seconde. En d'autres termes, cette requête peut être traitée à une vitesse d'environ plusieurs milliards de lignes par seconde sur un seul serveur. Cette vitesse est effectivement atteinte dans la pratique.

### CPU {#cpu}

Puisque l'exécution d'une requête nécessite le traitement d'un grand nombre de lignes, cela aide de répartir toutes les opérations sur des vecteurs entiers au lieu de lignes séparées, ou d'implémenter le moteur de requête de sorte qu'il n'y ait presque aucune surcharge de répartition. Si vous ne le faites pas, avec un sous-système de disque à moitié décent, l'interpréteur de requête bloque inévitablement le processeur. Il est logique de stocker des données dans des colonnes et de les traiter, si possible, par des colonnes.

Il y a deux façons de le faire:

1.  Un moteur vectoriel. Tous les calculs sont préparés pour des vecteurs, au lieu de valeurs séparées. Cela signifie que vous n'avez pas besoin de faire appel à ces calculs  très souvent, et les surcharges de répartition sont négligeables. Le code des calculs contient un cycle interne optimisé.

2.  La génération de Code. Le code généré pour la requête contient tous les appels indirects.

Ce n'est pas fait dans les bases de données “standard”, car cela n'a pas de sens lors de l'exécution de requêtes simples. Cependant, il y a des exceptions. Par exemple, MemSQL utilise la génération de code pour réduire la latence lors du traitement des requêtes SQL. (À titre de comparaison, les SGBD analytiques nécessitent une optimisation du débit, et non de la latence.)

Notez que pour l'efficacité du processeur, le langage de requête doit être déclaratif (SQL ou MDX), ou au moins un vecteur (J, K). La requête ne doit contenir que des boucles implicites, permettant une optimisation.

{## [Article Original](https://clickhouse.tech/docs/en/) ##}
