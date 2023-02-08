---
sectionid: lab2-service-invocation
sectionclass: h2
title: Invocation de Service
parent-id: lab-2
---

### Généralités

> **Question généraliste**: Qu'est-ce que le _Service Meshing_ ? Quels sont les exemples de logiciels proposants cette fonctionnalité ?

Solution :
{% collapsible %}
La meilleure manière de définir le _Service Meshing_ est de présenter le problème qu'il adresse.

Prenons l'exemple de deux services, A et B. A veut appeler B via HTTP.

Pour pouvoir contacter B, A va avoir besoin d'une information sur sa localisation. Cette information peut prendre la forme d'une URL, d'une adresse ip, ou même d'un couple (nom, namespace) pour Kubernetes par exemple.

Dans tous les cas, en fournissant cette information à A, nous couplons A et B. En effet, si B change de localisation, A devra aussi être mis à jour pour être fonctionnel.

C'est un problème important dans le monde des systèmes distribués, les services pouvant changer de localisation assez souvent que ce soit en terme de _noeud_, _namespace_, _url_...

Pour adresser ce problème, des services comme [_Istio_](https://istio.io/) ou [_Open Service Mesh_](https://openservicemesh.io/) sont apparus. Le principe de ces deux logiciels est de garder un catalogue de service associant le nom à une localisation, ajoutant donc une couche d'abstraction supplémentaire. Dans cette configuration, il est possible d'appeler les services par leur nom sans révéler leur localisation, _Istio_ ou _Open Service Mesh_ étant les seuls à la connaître et s'occupant de router l'appel.

Appliqué à notre exemple, un scénario **grandement simplifié** pourrait être le suivant :

L'instance d'_Istio_ pourrait contenir:

| Nom | Localisation            |
| --- | ----------------------- |
| A   | http://A.localapi.net   |
| B   | https://B.vendorapi.net |

A pourrait alors appeler B en utilisant uniquement son nom, avec une adresse de la forme `http://B.local`.
_Istio_ s'occuperait alors de router cet appel vers la véritable adresse de B, `https://B.vendorapi.net`.

Cette addresse `http://B.local` restera valide même si l'adresse réelle de B change, et seulement l'instance _Istio_ devra être mise à jour.
{% endcollapsible %}

> **Question généraliste**: Qu'est-ce que le _gRPC_ ? Quels sont ses avantages et inconvénients ?

Solution :
{% collapsible %}
Le _gRPC_ (Goog) est un framework crée par Google encapsulant des appels de procédures distantes - _Remote Procedure Calls_ (RPC).

Au contraire d'une API REST, qui se focalise sur la récupération de resources à l'aide d'une syntaxe unifiée, le _RPC_ est orienté vers des **actions**.
La manière la plus simple de s'imaginer le _RPC_ est d'imaginer appeler des function sur un ordinateur distant.

Une particularité du _gRPC_ par rapport aux autres framework de _RPC_ est qu'il utilise HTTP/2 et le langage de description [_Protobuf_](https://fr.wikipedia.org/wiki/Protocol_Buffers) pour décrire le contenu des messages.

C'est de cette particuliarité que proviennent les avantages du _gRPC_, la vitesse accrue et la taille diminuée des contenus envoyés.

Les inconvénients principaux sont :

- le _gRPC_ est bien plus difficile à interpréter pour un humain que le REST, en faisant une solution plutôt préférée pour la communication service de backend à service backend
- le temps de développement _gRPC_ est plus long
  {% endcollapsible %}

### Dapr

A l'aide de la [documentation](https://docs.dapr.io/developing-applications/building-blocks/service-invocation/service-invocation-overview/), nous allons nous intéresser à ces questions

> **Question** : Quel est le principe de l'invocation de service ? Quel est le trajet d'un paquet envoyé par le service A invoquant le service B ? Quel sont les protocoles utilisés lors des différentes étapes d'un trajet de paquet d'un service A a un service B ?

Solution :
{% collapsible %}
Le principe de l'invocation de service est de pouvoir invoquer une méthode d'un service distant de manière sécurisée et résiliente. Invoquer un service permet également à Dapr de générer automatiquement les logs et les traces.

Un paquet allant du service A au service B aurait donc le trajet:

```sh
A ---HTTP/gRPC---> sidecar de A
#  URL localhost:3500/invoke/B/method/order

sidecar de A ---HTTP/gRPC---> DNS server
#  B A ? (demande l'adresse ipv4 de B)

DNS server ---HTTP/gRPC---> sidecar de A
#  B A XXX.XXX.XX.XX

sidecar de A ---gRPC---> sidecar de B ---HTTP/gRPC---> B
#  Transmission de l'appel vers la méthode '/order' de B

```

{% endcollapsible %}

> **Question** : Dans [l'exemple de la page de la documentation](https://docs.dapr.io/developing-applications/building-blocks/service-invocation/service-invocation-overview#example), l'URL permettant à **pythonapp** d'appeler **nodeapp** est _http://localhost:3500/v1.0/invoke/nodeapp/method/neworder_. Décomposez cette URL et expliquez ses composantes.

Solution:

{%collapsible %}
En prenant chacune de ses composantes :

- **http://localhost:3500** : Appel vers le sidecar (non chiffré, inutile car même sandbox applicative)
- **v1.0** : Version de l'API Dapr
- **invoke** : Utilisation de l'API d'invocation de service
- **nodeapp** : Nom du service à appeller
- **method** : Appel d'une méthode sur le service
- **neworder** : Nom de la méthode à appeler sur le service
  {% endcollapsible %}

> **Question**: Quelle est la différence entre un _Service Meshing_ et l'invocation de service de Dapr ?

Solution:

{%collapsible %}
<u>Sur la cible</u>:
Le Service Meshing se déploie sur une infrastructure, et est unique à cette infrastructure. C'est une fonctionnalité OPS.
L'invocation de service de Dapr est indépendante de l'infrastructure, elle concerne le DEV.

<u>Sur les fonctionnalités</u>:
Si un Service Meshing et l'invocation de Service de Dapr permettent tous les deux de faciliter les appels de service à service, le service meshing va travailler au niveau du réseau, tandis que Dapr va travailler au niveau de l'application. Il en suit :

- Dapr **ajoute** une découverte des méthodes de l'application en plus de résoudre le nom du service
- Dapr **ne permettra pas** de redirections réseau à travers Internet (ou un tunnel) dans un cas d'application multi-clouds par exemple.

Il est donc possible d'utiliser un service comme _Istio_ en conjonction avec Dapr, les services n'ayant pas la même couverture fonctionnelle.

Voir https://docs.dapr.io/concepts/service-mesh/
{% endcollapsible %}

> **Ouverture** : Dans la page, il est fait mention de _Dapr Sentry_. Quel est son rôle ? Observez le fichier `docker-compose.yml` de l'exercice précédent, Sentry est-il présent ? Qu'en déduisez vous ?

Solution:

{%collapsible %}
Sentry permet le **chiffrage** et **l'authentification mutuelle** des communications entre services. Il permet la communication mTLS entre les services, agissant alors en tant que stockage / broker de certificats.

Sentry est un service totalement optionnel. S'il n'est pas présent au démarrage des *sidecars* (et son adresse spécifiée dans la commande de démarrage), les communications ne seront simplement pas chiffrées.

Dapr possède donc une architecture modulaire, et il existe d'autres services optionnels :

- **[Placement](https://docs.dapr.io/concepts/dapr-services/placement/)**, permettant l'utilisation du [modèle Acteurs](https://fr.wikipedia.org/wiki/Mod%C3%A8le_d%27acteur)
- Le **[Name Resolution Component](https://docs.dapr.io/reference/components-reference/supported-name-resolution/)** interne à Dapr est aussi modulaire. Par défaut, une résolution plate ([mDNS](https://en.wikipedia.org/wiki/Multicast_DNS)) est utilisée, mais **coreDNS**(Kubernetes) ou bien **Consul** peuvent également être utilisés selon les plateformes.
  {% endcollapsible %}

### En application

> **Note** : La nouvelle version de l'application se trouve désormais dans `src/Lab2/2-service-invocation`

Il est l'heure de reprendre le fil rouge. Cherchant toujours à rendre notre application de pré-commandes complète, deux nouveaux services ont été ajoutés:

- **stock-manager** (en Go): Une fois une commande validée,  **order-processing** appelle la méthode _/stock_ de **stock-manager** pour qu'il rajoute la commande aux stocks requis.
- **receipt-generator** (en Rust): Une fois une commande validée, **order-processing** appelle la méthode _/_ de **receipt-generator** afin qu'il génère une confirmation

Le nom de chaque service est également son app-id.

La nouvelle cible est donc:

![Expected result](/media/lab2/service-invocation/step-2-service-invocation.png)

> **Question**: Quelle est l'URL que doit utiliser **order-processing** pour appeler **stock-manager** ? Expliquez.

**Indice** : L'API d'invocation de Dapr est disponible [ici](https://docs.dapr.io/reference/api/service_invocation_api/)

Solution :
{% collapsible %}
D'après la documentation, l'URL d'une invocation de service est de la forme :

```sh
PATCH/POST/GET/PUT/DELETE http://localhost:3500/v1.0/invoke/<appId>/method/<method-name>
```

où :

- **localhost:3500** est l'adresse du sidecar
- **invoke** est le préfixe de l'API d'invocation
- **\<appId\>** est l'id du service à appeler tel que déclaré par le l'option `--app-id` de la ligne de commande de Dapr
- **\<method-name\>** est le nom de la méthode à appeler sur le service distant

Dans ce cas précis, le service à appeler est **stock-manager**, plus particulièrement la méthode _/stock_

L'URL recherchée est donc :

```sh
http://localhost:3500/v1.0/invoke/stock-manager/method/stock
```

{% endcollapsible %}

> **Question**: Quelle est l'URL que doit utiliser **order-processing** pour appeler **receipt-generator** ? Expliquez.

Solution :
{% collapsible %}
D'après la documentation, l'URL d'une invocation de service est de la forme :

```sh
PATCH/POST/GET/PUT/DELETE http://localhost:3500/v1.0/invoke/<appId>/method/<method-name>
```

où :

- **localhost:3500** est l'adresse du sidecar
- **invoke** est le préfixe de l'API d'invocation
- **\<appId\>** est l'id du service à appeler tel que déclaré par le l'option `--app-id` de la ligne de commande de Dapr
- **\<method-name\>** est le nom de la méthode à appeler sur le service distant

Dans ce cas précis, le service à appeler est **receipt-generator**, plus particulièrement la méthode _/_

L'URL recherchée est donc:

```sh
http://localhost:3500/v1.0/invoke/receipt-generator/method/
```

{% endcollapsible %}

> **En pratique**: A l'aide des deux questions précédentes, renseignez les variables d'environnements **RECEIPT_GENERATOR_INVOKE_URL** et **STOCK_MANAGER_INVOKE_URL** dans `docker-compose.yml`. Exécutez le fichier docker-compose et lancer une pré-commande via l'interface web (localhost:8089).

**Rappel**: Pour lancer un fichier docker-compose, lancez la commande suivante 

```sh
docker compose rm -fsv ; docker compose up --no-attach redis
```

La trace de succès devrait avoir cette forme :
![Expected result](/media/lab2/service-invocation/expected-result.png)

{% collapsible %}
Solution:

```diff
  ############################
  # Order Processing
  ############################
  order-processing:
    image: dockerutils/order-processing
    environment:
-      - STOCK_MANAGER_INVOKE_URL=
+      - STOCK_MANAGER_INVOKE_URL=http://localhost:3500/v1.0/invoke/stock-manager/method/stock
-      - RECEIPT_GEN_INVOKE_URL=
+      - RECEIPT_GEN_INVOKE_URL=http://localhost:3500/v1.0/invoke/receipt-generator/method/
```

{% endcollapsible %}

**Note** : Il est également possible de limiter quel(s) services(s) peuvent appeler un service. Pour cela un object configuration existe qui peut être passé en argument de chaque sidecar avec le switch de CLI **--config**. Voici un exemple où seul **order-processing** aurait le droit d'appeler le microservice.

```yml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  accessControl:
    defaultAction: deny
    policies:
      - appId: order-processing
        defaultAction: allow
```

Pour appliquer cette configuration à **receipt-generator** par exemple il faudrait:

- Créer un dossier config et y enregistrer la configuration ci-dessus (avec pour exemple le nom `config.yml`)
- Modifier le docker-compose comme suit:

```diff
  ############################
  # Receipt Generator
  ############################
  receipt-generator:
    image: dockerutils/receipt-generator
    environment:
      - RUST_LOG=debug
  receipt-generator-dapr:
    image: "daprio/daprd:edge"
    command: ["./daprd",
     "-app-id", "receipt-generator",
     "-app-port", "8081",
+    "-config /config/config.yml"
     "-components-path", "/components"]
    volumes:
        - "./components/:/components"
+       - "./config/config.yaml"
    network_mode: "service:receipt-generator"
```
