---
sectionid: first-app
sectionclass: h2
title: Une première application
parent-id: lab-1
---

> **Note** : Les fichiers utilisés dans cette première application se trouvent dans le dossier `src/Lab1/1-decoupling-direct`

### Une première application

Le support de cette première activité sera l'application représentée ci-dessous

![Première app](/media/lab1/first-app-vanilla.png)

Cette application est constituée de trois parties:

- Une instance de Redis pour permettre un stockage d'état.
- Un service Node qui stocke un état dans Redis
- Un service Python qui génère et envoie un état au service Node toutes les secondes

### Démarrer l'application en local

L'application peut être executée en local à l'aide de docker-compose.

```shell
    # L'option "--no-attach" permet de cacher les logs de redis
    # En cas de problème, elle peut être enlevée pour obtenir plus de détails
    docker-compose up --no-attach redis
```

> **En pratique** : Démarrez l'application. Vérifiez que vous obtenez une trace de cette forme:

![Résultats](/media/lab1/first-app-vanilla-result.png)
