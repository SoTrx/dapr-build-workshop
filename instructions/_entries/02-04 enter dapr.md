---
sectionid: lab1-enterdapr
sectionclass: h2
title: Le role de Dapr
parent-id: lab-1
---

Le rôle de Dapr est de permettre une sorte de découplage architecturel. Au lieu de devoir lier les composants les uns aux autres – avec l'utilisation de bibliothèques spécifiques par exemple –  nous allons pouvoir ajouter un degré d'abstraction supplémentaire permettant de simplifier le code.

Plus important : ce découplage permet au développeur d'un service de se **décharger de la responsabilité de l'implémentation**.

Pour illuster cela, nous allons déployer une autre version de l'application présentée qui elle utilise Dapr. 

> **Note** : Cette nouvelle application est située dans le dossier `src/Lab1/1-decoupling-direct`

Cette nouvelle version est exécutable en utilisant la commande suivante:

```shell
# On ignore les logs des services mongo et redis pour éviter une pollution des logs
docker compose up --no-attach mongo --no-attach redis
```

> **Question**: Comparez les deux versions (avec et sans Dapr) du service NodeJS. Comment le service recupère t-il le message ?

Solution :

{% collapsible %}
Un appel HTTP générique est effectué sur le port 3500 **de l'interface localhost** du service. Ce port est celui utilisé par défaut par Dapr.

```ts
// src/Lab1/1-decoupling/withDapr/node/src/app.ts, line 9
const daprPort = env.DAPR_HTTP_PORT || 3500;
const stateStoreName = `statestore`;
const stateUrl = `http://localhost:${daprPort}/v1.0/state/${stateStoreName}`;
```

En effet, Dapr fonctionne avec le principe du `sidecar`. Un sidecar est un petit programme qui va venir se greffer sur chacun des services que nous allons déployer. Chaque couple (application principale, sidecar) partage la même interface localhost.

À chaque fois que l'application principale voudra effectuer une communication vers un autre service, elle pourra se contenter d'appeller son sidecar, qui lui se chargera à son tour de transmettre l'appel

{% endcollapsible %}

> **Question** : Dans le fichier `docker-compose.yml`, où sont les *sidecars* Dapr ? Quelle est l'instruction docker-compose leur permettant à chaque *sidecar* d'accéder à l'interface localhost du service auquel il est attaché ? Le conteneur Redis a t-il un sidecar ?

Solution:
{% collapsible %}

```yml
  # Application principale
  nodeapp:
    build: ./node
  # Sidecar
  nodeapp-dapr:
    image: "daprio/daprd:edge"
    # Configuration du sidecar pour l'application node
    command: ["./daprd",
    # Id de l'application pour Dapr
    # Cet ID permet par la suite de faire de l'invocation de service (voir Lab2)
     "-app-id", "nodeapp",
    # Port d'écoute de l'application principale
    # Utilisé quand le sidecar transmet des informations à l'application principale
     "-app-port", "3000",
     # Niveau de verbosité du sidecar
     # "info" ou "debug" donneront plus de details sur le fonctionnement du sidecar
     "-log-level", "warn",
     # Où trouver les composants Dapr (question suivante)
     "-components-path", "/components"]
    volumes:
        - "./components/:/components:ro"
    # Demande au sidecar de partager son interface localhost avec l'application principale
    network_mode: "service:nodeapp"
```

Le conteneur Redis n'a pas de sidecar, puisque ce n'est pas un service mais seulement un moyen de communication (ou support de stockage). Il n'a pas besoin des fonctionnalités apportées par Dapr

{% endcollapsible %}

> **Question**: Toujours dans le `docker-compose.yml`, on peut remarquer qu'un [volume](https://docs.docker.com/storage/volumes/) nommé **components** est monté sur chacun des sidecars, que contient le dossier ? Quelle peut être l'utilité des fichiers qu'il contient ?

Solution:
{% collapsible %}

Ce dossier contient un fichier yaml ressemblant à ceci.

```yml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  # Parmi les composants de gestion d'état, on selectionne Redis
  type: state.redis
  version: v1
  # On renseigne l'adresse et les identifiants de connexion à Redis
  metadata:
    - name: redisHost
      # On peut résoudre l'ip de Redis par son nom
      # grâce au réseau hello-dapr de Docker-compose
      value: redis:6379
    - name: redisPassword
      value: ""
```

Il s'agit d'une définition d'un **[composant](https://docs.dapr.io/concepts/components-concept/)** Dapr. 

Ces composants sont une idée centrale de Dapr et ce sont eux qui permettent cette notion de découplage.
Ici, on annonce déclarativement que l'on veut utiliser Redis en tant que composant de stockage d'état.
Grâce à cette déclaration, tous les appels que les applications feront pour récupérer ou stocker un état seront
redirigés vers Redis.
Il suffirait de changer ce composant pour changer le support de stockage sous-jacent.

{% endcollapsible %}

Avec toutes ces informations en tête, nous allons pouvoir faire un récapitulatif.

![Première app avec Dapr](/media/lab1/first-app-dapr.png)

Le chemin parcouru par notre état est le suivant :

- Le service Python communique l'état à son sidecar avec une URL de la forme :

```bash
# On retrouve dans l'URL l'id spécifié dans le sidecar du service Node.
# Cette URL a pour but d'invoquer la méthode "neworder" sur le service "nodeapp"
# Nous verrons cela en détail dans le second lab
http://localhost:3500/v1.0/invoke/nodeapp/method/neworder
```

- Le sidecar du service python communique l'état au sidecar du service donc `-l'app-id` est celui spécifié dans l'URL, ici **nodeapp**
- Le sidecar du service Node transmet l'état au service Node
- le service node indique à son sidecar qu'il veut stocker un état dans le composant de stockage nommé **storename** avec un appel de la forme

```shell
POST /state/storename
```

- Dapr regarde alors ses composants et récupère le composant de gestion d'état de nom **storename**. Ce composant specifie l'hôte et les paramètres de connexion de l'implémentation du composant, ici Redis. L'appel est ensuite transféré vers Redis.

> **En Pratique** : En vous servant [de la documentation dediée](https://docs.dapr.io/reference/components-reference/supported-state-stores/setup-mongodb/), migrez le gestionnaire de message de Redis à MongoDB

**Indication** : La base de données MongoDB est déjà présente dans le fichier `docker-compose.yml`, il faut simplement l'utiliser

Solution:
{% collapsible %}
Pour changer le gestionnaire d'état de Redis vers MongoDB, il faut simplement changer le composant dédié, `src/Lab1/1-decoupling/withDapr/components/statestore.yaml`

En l'état le composant utilise Redis :

```yml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: pubsub
spec:
  type: pubsub.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis:6379
    - name: redisPassword
      value: ""
```

Changer simplement le type du composant en MongoDB et les identifiants d'accès permet de changer de gestionnaire d'état:

```yml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.mongodb
  version: v1
  metadata:
    - name: host
      value: mongo:27017
    - name: username
      value: root
    - name: password
      value: example
    - name: databaseName
      value: admin
```

{% endcollapsible %}
