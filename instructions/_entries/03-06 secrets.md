---
sectionid: lab2-secrets
sectionclass: h2
title: La gestion des secrets
parent-id: lab-2
---

Dans tous les scénarios que nous avons vu jusque là, nous avons par simplicité utilisé des services sans aucune forme d'authentification.
Dans un scénario de production, il sera néanmoins essentiel de s'adonner à une bonne gestion des secrets.

### Généralités

> **Question généraliste** : Qu'est-ce qu'un coffre fort numérique (ou gestionnaire de secrets) ? Quel est l'avatange de stocker des secrets dans ce coffre en lieu et place de les renseigner directement dans l'environnement des services ?

Solution :

{% collapsible %}

Un gestionnaire de secrets est un service centralisant la creation/récupération/suppression des secrets d'une application distribuée.

Bien qu'ajoutant une indirection supplémentaire, cette solution présente des avantages indispensables comme :

- Empêcher la duplication d'un secret utilisé plusieurs fois
- Permettre une forme d'authentification/autorisation, contrôlant quels services accèdent à quels secrets
- Garder une trace des accès, facilitant les audits de sécurité

{% endcollapsible %}

### Dapr

A l'aide de la [documentation](https://docs.dapr.io/developing-applications/building-blocks/secrets/secrets-overview/), répondez aux questions suivantes :

> **Question** : Quels sont les différents coffres forts supportés par Dapr ? Quels sont ceux utilisables en production ?

Solution :

{% collapsible %}

D'après [la documentation](https://docs.dapr.io/reference/components-reference/supported-secret-stores/)(Février 2023), les différents coffres forts disponibles sont :

| Nom                                   | Composant | Localisation              | Production ?                                                           |
| ------------------------------------- | --------- | ------------------------- | ---------------------------------------------------------------------- |
| Hashicorp Vault                       | Stable    | Externe / Dans le cluster | Oui                                                                    |
| Secrets Kubernetes                    | Stable    | Dans le cluster           | A éviter. Secrets en base64, limités à un namespace                    |
| Variables d'environnement             | Stable    | Dans le noeud             | Non.                                                                   |
| Fichier                               | Stable    | Dans le conteneur         | Non.                                                                   |
| Alibaba Parameter Store               | Alpha     | Externe                   | Oui. Attention à la maturité du composant                              |
| AWS Secrets Manager / Parameter Store | Alpha     | Externe                   | Oui. Attention à la maturité du composant                              |
| GCP Parameter Store                   | Alpha     | Externe                   | Oui. Attention à la maturité du composant                              |
| Azure Key Vault                       | Stable    | Externe                   | Oui                                                                    |

De manière générale, il sera toujours préférable de stocker les secrets en dehors de la plateforme d'exécution des services, pour ne pas "placer tous ses oeufs dans le même panier". Dans le cas où le gestionnaire de secret est hébergé sur la même plateforme que les services, il faudra prêter attention à la fiabilité de cette partie ultra critique (mode cluster, persistence, sauvegardes...).

{% endcollapsible %}

> **Question** : Quels sont les deux moyens pour accéder au gestionnaire de secrets à travers Dapr ?

Solution :

{% collapsible %}
La page de documentation présente deux moyens d'accéder au gestionnaire de secrets à travers Dapr:

- Utiliser l'API REST / les SDKs de Dapr depuis les **services** eux-mêmes
- Utiliser des références aux secrets depuis les **composants** déclarés

#### Depuis les services

Depuis le code des services, il suffit d'utiliser l'API correspondante :

```curl
GET/POST http://localhost:3500/v1.0/secrets/<NOM_COMPOSANT>/<NOM_SECRET>
```

où:

- **\<NOM_COMPOSANT\>** : Nom du composant de gestion de secrets
- **\<NOM_SECRET\>** : Clef / Namespace (en fonction du stockage sous-jacent) du secret stocké

#### Depuis les composants

L'autre manière est de modifier les composants Dapr existant pour y intégrer des références au gestionnaire de secret.

```diff
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    # Au lieu d'utiliser une valeur 
-    value: ""
    # Une référence est passée
+    secretKeyRef:
+    	name: <NAMESPACE>
+     key:  <SECRET_KEY>
+auth:
+  secretStore: <SECRET_STORE_COMPONENT>

```

{% endcollapsible %}

> **Question** : Proposez une configuration Dapr du service **service-1** lui permettant seulement d'accéder au secret **secret-1** dans le composant de gestion de secrets **vault**.

Solution :

{% collapsible %}

Il faudrait simplement appliquer cette configuration (voir partie monitoring) au service **secret-1**.

```yml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  secrets:
    scopes:
      - storeName: vault
        defaultAccess: deny
        allowedSecrets: ["secret-1"]
```

Il faut cependant noter les autres services auraient accès à l'ensemble des secrets, ce qui pourrait poser problème.

Une autre configuration possible dans ce cas serait d'appliquer une configuration à tous les services qui par défaut refuse l'accès à tous les secrets. De manière analogue au fonctionnement d'un pare-feu, il faudrait ensuite explicitement autoriser chaque service à accéder à chaque secret.
{% endcollapsible %}

### En application

L'application fil rouge continue alors à évoluer. Cette fois-ci, l'instance de Redis servant à la communication entre `command-api` et `order-processing` a été modifiée pour nécessiter un mot de passe : **suchStrongP4ssword!!**

Un nouveau service a également été ajouté dans l'application : un [Vault d'HashiCorp](https://www.vaultproject.io/). Ce coffre fort a été initialisé (en [mode de développement](https://www.vaultproject.io/docs/concepts/dev-server)) comme suit :

| Namespace | Valeur                          |
| --------- | ------------------------------- |
| redis     | REDIS_PASS=suchStrongP4ssword!! |

Ce coffre fort est accessible dans l'application à l'adresse `http://vault:8200`. Le jeton d'accès (_root token_) est **roottoken**.

> **En pratique**: A l'aide des informations ci-dessous, déclarez le composant Dapr (dans le dossier `src/Lab2/7-secrets/components`) correspondant au Vault.

**Indication** : Pensez à désactiver la vérification du TLS

Solution :

{% collapsible %}

Le coffre fort utilisé étant un Vault d'HashiCorp, il faut utiliser le [composant Dapr correspondant](https://docs.dapr.io/reference/components-reference/supported-secret-stores/hashicorp-vault/).

Ce composant dispose d'un grand nombre de paramètres. Cependant, le déploiement du Vault dans ce cas étant très simple, seulement trois paramètres sont réellement requis.

Le composant à créer est donc :

```yml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  # Le composant Dapr de nom "vault" ...
  name: vault
spec:
  # est une instance de Vault de Hashicorp ...
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
    # disponible à l'adresse http://vault:8200 ...
    - name: vaultAddr
      value: "http://vault:8200"
    # qui n'utilise pas de TLS ...
    - name: skipVerify
      value: "false"
    # dont le mot de passe est "roottoken"
    - name: vaultToken
      value: "roottoken"
```

{% endcollapsible %}

Le composant étant créé, il est possible de récupérer les secrets des deux manières que nous avons abordées plus haut.

> **En pratique**: Modifiez le composant `pubsub.yml` pour inclure la récupération du mot de passe depuis le composant de gestion de secret. Vérifiez le fonctionnement en exécutant l'application située à `src/Lab2/7-secrets/docker-compose.yml`

**Indication** : En cas d'échec de récupération du mot de passe de l'instance de Redis, certains services vont simplement crasher.

Solution :

{% collapsible %}

```yml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-pub-sub
spec:
  type: pubsub.redis
  version: v1
  metadata:
    - name: redisHost
      value: redis:6379
    # Le mot de passe est un secret ...
    - name: redisPassword
      secretKeyRef:
        # se trouvant dans le namespace "redis"
        name: redis
        # et est la valeur de correspondant à la clef REDIS_PASS
        key: REDIS_PASS
# Dans le composant de gestion de secrets appelé "vault"
auth:
  secretStore: vault
```
{% endcollapsible %}

### Conclusion

La gestion des secrets est une facette si essentielle de la vie d'une application distribuée qu'elle doit être incluse dès sa conception.

Dans cet exemple, nous utilisons un gestionnaire de secret directement intégré dans l'application, via un Vault. Si cette configuration est amplement suffisante dans le cadre d'un exercice, elle n'est évidemment pas conseillée sous cette forme en production.

Dans une configuration de production entièrement privée, il faudrait ainsi prévoir une couche de persistance pour le gestionnaire de secret et une meilleure sécurité d'accès.

Dans une configuration utilisant le Cloud public, il sera en revanche plus conseillé de se tourner vers un service managé, comme par exemple [Azure Keyvault](https://azure.microsoft.com/fr-fr/services/key-vault/).
