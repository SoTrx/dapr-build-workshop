---
sectionid: lab3-deploy-dapr-on-aca
sectionclass: h2
title: Deploy a Dapr app on ACA
parent-id: lab-3
---

Nous avons maintenant deployé une application sur ACA. Mais cette première application _hello-world_ n'utilise pas Dapr.

Nous allons donc reprendre l'application du Lab1 et la déployer sur ACA.

![Première app](/media/lab1/first-app-vanilla.png)

Hormis le statestore, cette application a besoin de deux services:

- L'application Node (disponible en tant qu'image à `dockerutils/nodeapp`)
- L'application Python (disponible en tant qu'image à `dockerutils/pythonapp`)

> **En pratique** : En utilisant la commande `az containerapp create`, créez 2 container app dans le même environnement pour les 2 services ci-dessus

**Indication** : Utilisez les images proposées pour les applications node et python, ce sera plus simple que de build les applications. Vous **n'avez pas besoin d'identifiants pour le registre de conteneurs**, il accepte les pull anonymes

Solution:

{% collapsible %}

```bash
# Nodeapp
az containerapp create \
  --name nodeapp \
  --image dockerutils/nodeapp \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --ingress 'internal' \
  --min-replicas 1 \
  --target-port 3000 \
  --enable-dapr \
  --dapr-app-id 'nodeapp'

# Pythonapp
az containerapp create \
  --name pythonapp \
  --image dockerutils/pythonapp \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --min-replicas 1 \
  --enable-dapr \
  --dapr-app-id 'pythonapp'
```

{% endcollapsible %}

> **Question**: Regardez les logs du conteneur **nodeapp**, que constatez-vous ? Proposez une explication.

Solution
{% collapsible %}

En regardant les logs de l'application node, on remarque qu'il y a **une erreur**.

![Trace](/media/lab3/trace-q1.png)

Cette erreur est due au fait que le composant Dapr `statestore` n'est pas déclaré.

{% endcollapsible %}

### Les composants Dapr sur Container Apps

Sur la version préliminaire de Container Apps ayant servi a rédiger ce workshop, la syntaxe des composants Dapr n'est pas encore supportée entièrement et il faut une adaptation légère des fichiers yml.

Par exemple pour le statestore Redis :

```yml
# Syntaxe DAPR
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
    - name: redisHost
      value: <HOTE>
    - name: redisPassword
      value: ""
```

```yml
# Syntaxe ACA
componentType: state.redis
version: v1
metadata:
  - name: redisHost
    value: <HOTE>
  - name: redisPassword
    value: pass
# A quelles applications s'applique ce composant
scopes:
  - nodeapp
```

Une autre limitation de la préversion est que certains composants ne sont pas totalement supportés. C'est le cas du composant redis.

Pour stocker l'état de l'application, nous allons donc utiliser un **Storage Account** sur Azure.

Pour créer un nouveau Storage Account exécutez la commande suivante

```bash
#N'oubliez pas de remplacer les variables, et que le nom du storage acount ne doit pas contenenir de caractères
# speciaux ou d'espaces
az storage account create --name $STORAGE_ACCOUNT --resource-group $RESOURCE_GROUP --location westeurope --sku Standard_LRS --kind StorageV2

#Ensuite, on recupère la clef d'accès au storage account crée
az storage account keys list --resource-group $RESOURCE_GROUP --account-name $STORAGE_ACCOUNT --query '[0].value' --out tsv
```

Ensuite créez un nouveau fichier yml pour le composant de stockage d'état Dapr sur ce Storage account

```yml
componentType: state.azure.blobstorage
version: v1
metadata:
  - name: accountName
    value: "<STORAGE_ACC_NAME>"
  - name: accountKey
    secretRef: account-key
  - name: containerName
    value: state
secrets:
  - name: account-key
    value: "<STORAGE_ACC_KEY>"
scopes:
  - nodeapp
```

Pour forcer la prise en compte du composant, on redéploie **nodeapp**

```bash
az containerapp create \
  --name nodeapp \
  --image dockerutils/nodeapp \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --ingress 'internal' \
  --target-port 3000 \
  --enable-dapr \
  --dapr-app-id 'nodeapp'
```

> **En pratique** : Allez à nouveau regarder les logs de l'application Node pour vérifier le fonctionnement.

> **Pour les plus avancés (Optionnel)** : Déployez l'applicaton du Lab2 sur ACA.

Et c'est sur ce dernier exercice que se termine ce workshop
