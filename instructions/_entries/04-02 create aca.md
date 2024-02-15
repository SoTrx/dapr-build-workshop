---
sectionid: lab3-create-aca
sectionclass: h2
title: Create a new ACA environment
parent-id: lab-3
---

Pour créer votre prmière application Container Apps nous allons utiliser la CLI, plus particulièrement une des commandes de l'extension qui a été installée en pré-requis `az containerapp create`.

Mais avant de créer une application, il faut tout d'abord créer un nouvel environnement Container Apps.

**Explication**:
Un **environnement** Containers Apps est un groupe dans lequel sont publiés des services. Tous les services publiés dans le même environement partage le même réseau (vNet), le même aggrégateur de logs (workspace Logs Analytics) et éventuellement les mêmes ressources de calcul sous-jacentes (si l'orchestrateur les affectes sur la même machine).
Pour que deux services puissent communiquer avec Dapr, ils doivent partager le même environnement.
![Aca env explanation](/media/lab3/create-aca-env-explanation.png)

Pour créer un nouvel environnement, les commandes suivantes peuvent être utilisée :

```bash
az group create -n <Nom_ressource_group>  --location westeurope
az containerapp env create -n <Nom_env> -g <Nom_ressource_group> --location westeurope
```

Notre environnement étant crée, il ne nous reste plus qu'à créer les services à l'intérieur !
