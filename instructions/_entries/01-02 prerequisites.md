---
sectionid: prereq
sectionclass: h2
title: Prérequis
parent-id: intro
---

### Prérequis

Ce workshop va demander les éléments suivants:

- une souscription Azure
- Azure CLI (**>= 2.30**) **et son extension** pour Containers Apps
- [Docker](https://www.docker.com/) avec son extension [compose](https://docs.docker.com/compose/install/)(**>=1.27.0**). L'extension est incluse dans Docker-Desktop, il faudra donc seulement l'installer sur Linux.
- [Les sources du workshop](https://aka.ms/daprartifacts)

### Installer Azure CLI et l'extension pour Azure Container Apps

#### Si CLI non installée : Installer le CLI

Suivez [ce lien](https://docs.microsoft.com/fr-fr/cli/azure/install-azure-cli) et suivez l'onglet correspondant à votre système d'exploitation.

#### Si CLI installée : Upgrade sa version du CLI

```bash
az version
# Si version az-core <= 2.30
az upgrade
```

#### Installer l'extension Azure Container Apps

Une fois la CLI installé, installez l'extension de la CLI pour Azure Container Apps

```bash
az extension add -n containerapp
```

#### Se connecter à sa souscription

Puis enfin connectez vous à votre souscription à l'aide de la commande suivante

```bash
az login
```
