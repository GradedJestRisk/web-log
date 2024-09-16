---
layout: post
title:  "Passer une PK en BIG INTEGER"
date:   2021-08-30 10:10:42 +0200
categories: bdd
excerpt: "Changer le type d'une clef primaire de INTEGER à BIG INTEGER en minimisant l'indisponibilité"
authors: pierre_top
---

## TL;DR

Notre table la plus volumineuse aurait atteint son remplissage maximal dans quelques mois à cause du type de la clef primaire. Pour éviter cela, nous souhaitions changer de type en minimisant l'impact utilisateur, en visant des migrations de BDD "zero-downtime".

Nous avons réussi (pour les impatients, la solution est [ici](##implementation)) et partageons avec vous nos apprentissages.

Ceci dit, cela a eu lieu dans des circonstances particulières :

- table non référencée par d'autres tables;
- traffic réduit (vacances).

Ce chantier continuera sur une autre PK, sans ces deux circonstances, et nous partagerons également les résultats dans un autre post.

## Motivation

Le propos est peut-être un peu appuyé, mais il a le mérite d'être clair.

> Friends don't let friends use INT as a primary key.
<https://github.com/rails/rails/pull/26266#issue-82461683>

En effet, si vous générez des identifiants pour servir de clef primaire, et si pour cela utilisez la séquence par défaut, alors :

- elle commence à 1 ;
- son type est [INTEGER](https://www.postgresql.org/docs/current/datatype-numeric.html), et s'arrête à 2 milliards (2^16/2 pour les puristes).

Si vous avez plusieurs relations 1/N en cascade, cela peut être atteint "rapidement", par exemple :

- 1 centrale d'achat de grande distribution ;
- 20 00 clients (point de vente) ;
- 2 000 commandes par client (10 ans de commandes journalières) ;
- 1000 lignes par commande.

Il est bien sûr possible de modifier le type de cette propriété avec l'instruction native `ALTER COLUMN TYPE`, mais cela a un coût proportionnel au remplissage de la table.

Dans le contexte Pix, les développeurs qui ont choisi d'utiliser le type par défaut ne pouvaient pas savoir que la plateforme rencontrerait un tel succès.

Bref, si comme nous, vous avez une table dans ce cas, voyons comment :

- effectuer ce changement [avant qu'il ne soit trop tard](https://twitter.com/dhh/status/1060565296048562177?lang=fr) ;
- de la manière la plus fluide possible.

## Recherche de solution

La recherche préalable est bien résumée par [https://stackoverflow.com/questions/54795701/migrating-int-to-bigint-in-postgressql-without-any-downtime/54796046](ce thread) Stack Overflow qui propose deux solutions :

- utiliser une colonne temporaire ;
- utiliser une base de données temporaire, alimentée par une réplication logique.

La solution 2 est rapidement écartée. Nous utilisons les services d'un [PaaS](https://scalingo.com/) et sommes heureux de ne pas avoir à installer et maintenir nos bases de données. Le service spécifique de la réplication logique n'est pour l'instant pas offert.

### Etape 1 : convertir les données en tâche de fond

La [solution 1](https://tech.coffeemeetsbagel.com/schema-migration-from-int-to-bigint-on-a-massive-table-in-postgres-aabb835c3b84) effectue en tâche de fond la conversion des données implicite dans `ALTER TABLE foo COLUMN idBigInteger BIG INTEGER`.

La [documentation](https://www.postgresql.org/docs/current/sql-altertable.html#notes) mentionne
> Changing the type of an existing column will require the entire table and its indexes to be rewritten.
> As an exception, when changing the type of an existing column and the old type is either binary coercible to the new type a table rewrite is not needed; but any indexes on the affected columns must still be rebuilt.
> Table and/or index rebuilds may take a significant amount of time for a large table; and will temporarily require as much as double the disk space.

Il pourrait sembler qu'aucune conversion n'est nécessaire de `INTEGER` à BIGINTEGER`, mais ce n'est pas le cas.
```sql SELECT 1::integer::bigint```

Comme l'opération native pose un verrou exclusif sur la table, il faut attendre sa réécriture complète pour y accéder.
> An ACCESS EXCLUSIVE lock is held unless explicitly noted.

### Etape 2: substituer les colonnes et créer la clef primaire

Une fois la migration achevée, une fenêtre de maintenance est requise (1h dans son cas) pour :

- substituer la colonne temporaire ;
- valider les contraintes (PK et FK).

### Améliorations

J'en parle avec [Jonathan](https://github.com/jonathanperret). Tout en validant la démarche, il pense que la fenêtre de maintenance est encore trop longue et suggère des modifications pour la réduire à quelques secondes.

Le principe est le suivant :

- la contrainte de clef primaire comporte une contrainte NOT NULL et une contrainte UNIQUE ;
- c'est la validation de ces contraintes qui prend le plus de temps dans la fenêtre de maintenance ;
- ces validations peuvent être effectuées en amont.

Pour valider les contraintes en amont :

- ajouter une contrainte NOT NULL dès la création de la colonne ;
- créer un index UNIQUE de manière concurrente [`CREATE INDEX CONCURRENTLY`](https://www.postgresql.org/docs/current/sql-createindex.html).

Lors de la création de la PK, il suffit de préciser l'index UNIQUE (PostgreSQL détecte la contrainte NOT NULL).
`ALTER TABLE "foo" ADD CONSTRAINT "foo_pkey" PRIMARY KEY USING INDEX "foo_id_unique`

## Implémentation

La solution est présentée en pseudo SQL/NodeJS sur une table imaginaire, avec les liens vers les commits réels.

On part de là

``` sql
CREATE TABLE foo (ID INTEGER PRIMARY KEY);
INSERT INTO foo generate_series(1, 1e6);
```

Voilà la solution native, avec accès exclusif sur la table pendant 8 heures.

```sql
ALTER TABLE foo COLUMN idBigInteger BIG INTEGER;
```

Tout commence avec une migration, lancée lors du déploiement de la release.
<https://github.com/1024pix/pix/pull/3357/files#diff-1eecfd94150cca766655d3786c7dc12e3b6bc8fa60be3a0c0740424ab02c157a>

```sql
ALTER TABLE foo ADD COLUMN idBigInteger NOT NULL DEFAULT -1;
CREATE TRIGGER WHEN INSERT INTO foo FOR EACH ROW :NEW.bigIntId = :new.id;
```

Ensuite, le script de migration inclus dans la PR est lancé manuellement (durée: 48h).
<https://github.com/1024pix/pix/pull/3357/files#diff-d89cb20637cb9b836bce4902b46db958e0209c2a0de96e7dc81a0c43015680b4>

```javascript
const maxid = await client.query('SELECT MAX(id) INTO maxId FROM foo');
for (const i = 0; i <= maxId; i++){
  await client.query(`UPDATE foo SET idBigInteger = id WHERE id BETWEEN i AND (i + ${chunkSize} -1)`);
}
await client.query('CREATE UNIQUE INDEX CONCURRENTLY indexidBigInteger');
```

Pour finir, une dernière migration lancée lors du déploiement de release
<https://github.com/1024pix/pix/pull/3364/files#diff-9985cafe7c9915c64927036180071c804f75aad26a66686efd1758d242b819d1>

```sql
BEGIN TRANSACTION;
LOCK foo WITH ACCESS EXCLUSIVE;
ALTER SEQUENCE seqPkFoo TYPE BIG INTEGER;
ALTER TABLE foo ALTER COLUM bigIntId SET DEFAULT seqPkFoo.nextval();
ALTER TABLE foo DROP COLUMN id;
ALTER TABLE foo RENAME COLUMN idBigInteger TO id;
ALTER TABLE foo ADD PRIMARY KEY id USING indexidBigInteger;
COMMIT TRANSACTION;
```

## Exécution en production et perspectives

### Le plan se déroule sans accroc

La migration préalable s'est déroulée sans erreur.

La migration des données proprement dite a pris environ 24h.

L'exposition de la nouvelle colonne, lors du déploiement, a pris un temps négligeable pour les utilisateurs finaux (< 1 seconde).

```shell
2021-08-27 14:10:40.368443479 +0200 CEST [postdeploy-2719] > pix-api@3.92.0 postdeploy /app
2021-08-27 14:10:40.368393446 +0200 CEST [postdeploy-2719]
2021-08-27 14:10:41.566685265 +0200 CEST [postdeploy-2719] Batch 188 run: 1 migrations
2021-08-27 14:10:42.176473412 +0200 CEST [manager] container [postdeploy-2719] (6128d63ffcff1013ac9e57c1) has stopped
```

### Mais

#### Un impact non négligeable du traitement de migration sur le temps de réponse

Bien que l'opération soit un succès, nous avons couru plusieurs risques :

- la migration des données (phase `UDPATE`) a sensiblement ralenti le temps de réponse global ;
- la migration des données (phase `CREATE INDEX CONCURENTLY`) a acquis un verrou exclusif sur une période de x minutes.

Grâce à des circonstances favorables, l'impact a été réduit :

- la migration a eu lieu en été, période de faible fréquentation de la plateforme ;
- le verrou exclusif a été acquis à un horaire de faible fréquentation (nuit).

Nous avions envisagé (et testé) des mesures de mitigation du risque, notamment :

- la possibilité d'arrêter/reprendre le traitement de migration ;
- la capacité de réduire l'impact du traitement (pauses périodiques, réduction de la taille des batchs).

Malgré cela, il reste un point faible dans cette approche, l'acquisition du verrou exclusif.

#### Un impact sur l'organisation des données sur le disque

Nous avons constaté que la table est bien plus fragmentée après migration qu'avant migration. Bien que nous n'ayons pas fait de test comparatif, il est possible que cela soit dû à notre solution.

En effet :

- la solution native réécrit la table en une seule fois ;
- notre solution la réécrit en plusieurs fois.

Cela peut avoir deux effets :

- occupation de disque plus importante ;
- impact sur les temps de réponse.

Nous monitorons actuellement les temps de réponse pour déterminer s'il y a un impact.

## Perspectives

Ces actions font partie d'un chantier qui a dans son périmètre d'autres tables.
Nous ne pourrons pas appliquer exactement la même approche à l'avenir, pour deux raisons qui seront détaillées. Aussi, nous allons continuer à y travailler et cela fera l'objet d'un nouvel article.

### Diminuer les impacts possibles

Le contexte de faible fréquentation de la plateforme ne peut pas être pris comme un pré-requis. En particulier, l'existence d'un verrou exclusif sur un période de plusieurs minutes est problématique.

Nous pensons à :

- modifier notre approche en passant d'une colonne temporaire à une table temporaire ;
- rendre le traitement de migration paramétrable à chaud, en passant de variables d'environnement à une table de paramétrage.

### Généraliser la solution

La table qui a été migrée n'était pas référencée par d'autres tables.
Cela était implicite, car elle était la plus volumineuse.

Cependant, comme le rappelle Gerald Weinberg avec "Rudy's Rutabaga rule"
> Once you eliminate your number one problem, number two gets a promotion.

Notre problème n°2 est une table d'une volumétrie inférieure à celle migrée, mais d'une volumétrie suffisante pour être concernée. Et cette table est elle-même référencée par la table migrée.

La situation est similaire, mais seule l'implémentation nous dira si cela est fondé:

- deux colonnes à mettre à jour au lieu d'une ;
- une contrainte de type FK au lieu d'une contrainte de type PK.

