---
sectionid: lab3-hello-world-aca
sectionclass: h2
title: A first app on ACA
parent-id: lab-3
---

### Hello world

Cette première application sera juste une application hello-world, le conteneur `mcr.microsoft.com/azuredocs/containerapps-helloworld:latest`. Cette application exposer **une petite interface web sur son port 80**.

> **En pratique**: En vous servant de la [documentation](https://docs.microsoft.com/fr-fr/cli/azure/containerapp?view=azure-cli-latest#az-containerapp-create), déployez le conteneur hello-world sur Azure Container Apps.

**Indication** : Pour obtenir les différents paramètres disponibles de la commande `az containerapp` vous pouvez utiliser la command suivante

```csharp
az containerapp --help
```

**Indication 2** : N'oubliez pas d'exposer votre conteneur sur internet en fixant l'option `--ingress` à la valeur `external`

Solution:
{% collapsible %}

```bash
# Il faudra enlever les "\" si vous utilisez PowerShell
az containerapp create \
  # Nom du conteneur à déployer
  --name my-container-app \
  # Source du conteneur, ici une image docker sur le registre de Microsoft
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  # Groupe de ressource dans lequel se trouve l'instance d'ACA (que vous avez du noter)
  --resource-group $RESOURCE_GROUP \
  # Environnement ACA dans lequel créer le conteneur (~ dans quel groupe de conteneur)
  --environment $CONTAINERAPPS_ENVIRONMENT \
  # Ce conteneur doit être exposé sur Internet.
  # Une valeur 'internal' aurait signifié que le conteneur doit uniquement être exposé aux autres
  # conteneurs de son environnement.
  --ingress 'external' \
  # Port du conteneur à exposer
  --target-port 80 \
  # Optionel, retourne uniquement le FQDN du conteneur une fois crée
  --query configuration.ingress.fqdn
```

Si vous n'avez pas spécifié de --query, le retour ressemblera à ça :

```jsonc
{
  // GUID du conteneur sur Azure
  "id": "/subscriptions/<subId>/resourceGroups/dapr-workshop/providers/Microsoft.App/containerApps/hello-world",
  // Identité (systme ou utlisateur) associée au conteneur
  "identity": {
    "type": "None"
  },
  // Region Azure sur laquelle est deployé le conteneur
  "location": "West Europe",
  // Nom du conteneur
  "name": "hello-world",
  "properties": {
    "configuration": {
      // Une seule version (revision) de l'application est active à la fois
      // Dans le cas contraire "Multiple", il serait possible d'avoir à la fois la version
      // v1 et v2 de l'application active en même temps, avec 80% du traffic sur la v1 et 20% sur la v2
      // Cette fonctionnalité permet un meilleur chemin de mise à jour
      "activeRevisionsMode": "Single",
      // Ingress gateway
      "ingress": {
        // Autoriser HTTP ?
        "allowInsecure": false,
        // Sur internet ?
        "external": true,
        // URL
        "fqdn": "hello-world.nicemushroom-f16f2874.westeurope.azurecontainerapps.io",
        // Port du conteneur visé
        "targetPort": 80,
        // Répartition du traffic
        "traffic": [
          {
            "latestRevision": true,
            "weight": 100
          }
        ],
        "transport": "Auto"
      }
    },
    "customDomainVerificationId": "...",
    "latestRevisionFqdn": "hello-world--iz02pfi.nicemushroom-f16f2874.westeurope.azurecontainerapps.io",
    "latestRevisionName": "hello-world--iz02pfi",
    "managedEnvironmentId": "/subscriptions/<sub-id>/resourceGroups/dapr-workshop/providers/Microsoft.App/managedEnvironments/sample-env",
    "outboundIpAddresses": [...],
    "provisioningState": "Succeeded",
    "template": {
      // Conteneurs déployés
      "containers": [
        {
          "image": "mcr.microsoft.com/azuredocs/containerapps-helloworld:latest",
          "name": "hello-world",
          "resources": {
            "cpu": 0.5,
            "memory": "1Gi"
          }
        }
      ],
      // L'autoscaling est activé de 1 à 10 par défaut
      "scale": {
        "maxReplicas": 10
      }
    }
  },
  "resourceGroup": "dapr-workshop",
  "systemData": {...},
  "type": "Microsoft.App/containerApps"
}
```

{% endcollapsible %}

Accédez au FQDN de l'application de démo. Vous devriez voir ce message.

![Running app](/media/lab3/running-app.png)

Dernière étape, dans le [portail Azure](https://portal.azure.com), naviguez jusqu'à l'application déployée. Pour cela naviguez jusqu'à l'intérieur de votre resource group et cliquez sur la container app _hello-world_ crée

> **En pratique**: Explorez ce que propose le portail Azure pour l'application _hello-world_. Comment trouver les métriques ? Les logs ?

Et avec cela, nous avons déployé notre première application dans Containers Apps.
