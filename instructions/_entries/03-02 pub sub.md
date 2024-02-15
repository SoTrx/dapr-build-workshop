---
sectionid: lab2-pubsub
sectionclass: h2
title: Pub Sub
parent-id: lab-2
---

### Généralités

> **Question généraliste** : Qu'est-ce que le pattern _Publish Subscribe (PUB/SUB)_ ? A quel besoin repond t-il ?

Solution:
{% collapsible %}
Le pattern publish/subscribe est un mode de communication asynchrone.

On va retrouver dans ce mode de communication deux types d'acteurs:

- des **Producteurs** (_Publishers_), qui produisent un contenu, souvent des messages
- des **Consommateurs** (_Subscribers_), qui attendent du contenu

Par analogie, si une invocation directe pourrait être considérée comme un appel téléphonique, le _pub/sub_ serait alors une boîte mail, que chacun pourrait consulter quand bon lui semble.

La majorité du temps, le pub/sub fonctionne en **_one to many_**, c'est à dire qu'un message d'un producteur va parvenir à tous les consommateurs. Il existe cependant des configurations **_one to one_**, où un message d'un producteur n'est reçu que par un seul consommateur.
{% endcollapsible %}

### Dapr

A l'aide de la [documentation](https://docs.dapr.io/developing-applications/building-blocks/pubsub/), nous allons nous intéresser à ces questions:

> **Question** : Comment fonctionne la fonctionnalité PUB/SUB de Dapr ? Quel est le chemin d'un message depuis son envoi d'un service A vers un service B ?

Solution :
{% collapsible %}
![Pub/Sub overview](/media/lab2/pubsub/pubsub-overview.png)
En reprenant l'image de la documentation, on peut voir que:

- le message passe du service à son sidecar
- le sidecar résoud le composant, et redirige le message vers l'implémentation sous-jacente
- l'implémentation notifie tous les sidecars avec le contenu du message
- les sidecars redirigent le contenu du message vers une route HTTP de leurs services respectifs
  {% endcollapsible %}

> **Question** : Quelle est la garantie de livraison associée à la fonctionnalité de PUB/SUB ? Quels sont les avantages et les incovénients ?

Solution :
{% collapsible %}
La garantie est **at least once**. ([Lien dans la documentation](https://docs.dapr.io/developing-applications/building-blocks/pubsub/pubsub-overview/#at-least-once-guarantee)).
Cette garantie signifie que chaque message envoyé par un producteur sera reçu au moins une fois par chaque consommateur.
L'avantage est qu'il n'est pas possible de perdre un message, tout du moins dès lors qu'au moins un consommateur est inscrit au moment de l'envoi du message.

L'inconvénient principal est qu'il est possible de recevoir plusieurs fois le même message.
Cet inconvénient peut se traiter applicativement. Un exemple est de retenir l'ID des X derniers messages traités, ou simplement d'avoir un service resistant par conception à la duplication.

##### Pour aller plus loin

Lors de la conception d'un système de communication, si la possibilité de duplication est absolument à éviter il faudra plutôt s'orienter vers une garantie **at most once**. Un message sera alors livré au plus une fois, ce qui évitera la duplictaion mais pourra occasionner des pertes de messages.

La garantie idéale, **exactly once**, n'est pas proposée dans le cas général. En effet, il existera toujours un cas limite où il faudra relancer la transaction (comsommateur qui échoue pendant la reception d'un message...).

{% endcollapsible %}

> **Question** : Il existe deux méthodes pour souscrire à un _topic_ dans Dapr, quelles sont-elles ? Dans quels cas utiliser les deux ?

Solution :

{% collapsible %}

Pour souscrire à un _topic_, Dapr propose deux méthodes:

##### Dans le code

C'est le moyen "classique". On utilise le SDK pour définir un callback quand un message est reçu.

```ts
import { DaprServer } from "@dapr/dapr";
const server = new DaprServer();
// Souscrire au topic "orders" sur le composant de nom "pubsub"
await server.pubsub.subscribe("pubsub", "orders", async (orderId) => {
  console.log(`Subscriber received: ${JSON.stringify(orderId)}`);
});
await server.startServer();
```

##### De manière déclarative

L'autre manière est de déclarer la souscription comme on le ferait avec un composant.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: order
spec:
  # Sur le composant de nom "pubsub"...
  pubsubname: pubsub
  # quand un message est reçu sur le topic "orders"...
  topic: orders
  # envoyer le contenu du message sur l'endpoint POST /checkout...
  route: /checkout
# des services "orderprocessing" et "checkout"
scopes:
  - orderprocessing
  - checkout
```

L'avantage est que les services "orderprocessing" et "checkout" pourront rester génériques. Ils n'auront qu'à utiliser un serveur HTTP écoutant sur l'endpoint /checkout pour recevoir le message.

{% endcollapsible %}

> **Ouverture** : Sur la documentation, une fonctionnalité mise en avant est le routage basé sur le contenu (_Content-Based Routing_). Quel est le principe de cette fonctionnalité ? A quel besoin repond t-elle ?

{% collapsible %}
Le routage basé sur le contenu permet à Dapr de choisir l'endpoint HTTP sur lequel sera redirigé le message en fonction du contenu du message.

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: myevent-subscription
spec:
  # Sur le composant de nom "pubsub"...
  pubsubname: pubsub
  # quand un message est reçu sur le topic "inventory"...
  topic: inventory
  # envoyer le contenu du message...
  routes:
    rules:
      # sur l'endpoint "/widgets" si le message est de type "widget"...
      - match: event.type == "widget"
        path: /widgets
      # sur l'endpoint "/gadgets" si le message est de type "gadget"...
      - match: event.type == "gadget"
        path: /gadgets
    # sur l'endpoint "/products" sinon
    default: /products
# sur les applications "app1" et "app2"
scopes:
  - app1
  - app2
```

Cette fonctionnalité est pensée dans les cas où l'appplication reçoit un grand nombre d'événements différents. Elle permet :

- d'empêcher de créer un grand nombre de topics, un pour chaque cas limite de l'application. Sur le Cloud public, cela résoud notamment une problématique de coût.
- d'éviter à l'application elle-même de faire le routage, ce qui entraînerait une complexité inutile.

Il faut cependant noter que l'abus de cette fonctionnalité pourrait entraîner une difficulté de compréhension des flux de l'application. La meilleure manière d'éviter une telle situation est de prévoir son utilisation dès l'étape de conception.
{% endcollapsible %}

### En application

Voici l'état actuel de l'application de pré-commande fil rouge :

![Step 0](/media/lab2/app-step-0.png)

Le frontend va être la vitrine de notre site, qui s'exécute sur le navigateur du client (raison pour laquelle il n'a pas de sidecar Dapr rattaché) et l'API la porte d'entrée de l'architecture du backend.

> **Note** : Cette nouvelle application se trouve dans le dossier `src/Lab2/1-pubsub`

Pour démarrer l'application, il suffit de lancer la commande

```shell
# docker compose rm permet d'être sûr que les changements dans docker-compose.yml
# sont pris en compte
docker compose rm -fsv ; docker compose up --no-attach redis
```

Le frontend devrait être disponible à l'adresse `localhost:8089`. Cliquer sur le boutton **Commander** permet de lancer une commande.

La prochaine étape dans le développement de cette application est de rajouter un service `order-processing` qui va recevoir et traiter les commandes.

Pour éviter de saturer le service en cas de forte affluence, nous allons opter pour un traitement asynchrone des commandes, en utilisant le pattern Pub/Sub (_Pas forcément idéal, voir Note 1_).

Voici donc la cible:

![Step 1](/media/lab2/pubsub/app-step-1-pubsub.png)

A noter:

- Chaque texte en violet est une variable d'environnement à remplir dans `src/Lab2/1-pubsub/docker-compose.yml`
  - **PUB_URL** : URL appelée par le service **command-api** pour publier un message. Il s'agit d'un appel vers son sidecar, il sera donc préfixé par _http://localhost:3500_.
  - **COMMAND_API_HOST** : URL du service **command-api** au regard de **command-frontend**. Cette valeur est déjà pré-remplie
- **/process-order** : Endpoint HTTP POST de traitement de message. Les nouveaux messages doivent être redirigés vers cette URL.

> **Question**: Dans le dossier `src/Lab2/1-pub-sub/components`, il y a maintenant un nouveau fichier `pubsub.yaml`. Quelle est son utilité ? Quel est le nom du **composant dapr** associé ?

Solution :

{% collapsible %}

Ce nouveau fichier est défini comme suit :

```yaml
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
    - name: redisPassword
      value: ""
```

Il s'agit du composant Dapr faisant le lien avec l'implémentation du pubsub, ici Redis.

Le nom du composant (la valeur de la clef du yaml **metadata.name**) est `order-pub-sub`.
Attention à ne pas confondre ce nom avec le nom du fichier qui lui est simplement pubsub.

{% endcollapsible %}

> **Question**: A l'aide du schéma ci dessus, identifier comment implémenter le PUB/SUB pour `command-api` et `order-processing`

**Indice** : L'URL qui permet à **command-api** de publier un message est un appel vers son sidecar. L'API pub/sub de Dapr est détaillée [ici](https://docs.dapr.io/reference/api/pubsub_api/).

**Indice 2** : Il n'est **pas demandé** de faire des modifications dans le code des services ! Souvenez-vous qu'il existe plusieurs manières de souscrire à un *topic*

Solution :

{% collapsible %}

Parmi ces deux services, **command-api** est le producteur. Comme les SDKs ne sont pas utilisés, il faut trouver l'appel correct vers son sidecar.

##### Command API

Le service **command-api** est le producteur. Pour pouvoir publier un événement, il a besoin de l'URL vers laquelle publier son contenu.

Cette URL peut se trouver dans la [documentation](https://docs.dapr.io/reference/api/pubsub_api/), et est de la forme :

```sh
POST http://localhost:<daprPort>/v1.0/publish/<pubsubname>/<topic>
```

où :

- localhost:3500 est l'adresse du sidecar
- publish est le préfixe de l'API pub/sub
- <pubsubname> est le nom du composant de pubsub à utiliser, ici **order-pub-sub**
- <topic> est le topic dans lequel publier le message. Cela peut être n'importe quelle valeur. Ici nous allons utiliser **orders**

Une fois les variables remplies, l'URL d'invocation est donc

```sh
http://localhost:3500/v1.0/publish/order-pub-sub/orders
```

Il ne reste plus qu'à renseigner cette URL dans les variables d'environnements du fichier `src/Lab2/1-pubsub/docker-compose.yml`

```diff
...
  ############################
  # Command API
  ############################
  command-api:
    image: dockerutils/command-api
    environment:
-     - PUB_URL=
+     - PUB_URL=http://localhost:3500/v1.0/publish/order-pub-sub/orders
...
```

##### Order processing

**Order-processing** est le consommateur. Il faut donc qu'il souscrive au topic **orders**, celui que **command-api** utilise pour cet exemple.

A l'aide des questions précédentes, nous savons qu'il existe deux méthodes pour souscrire à un topic :

- par le code
- déclarativement

Le choix ici est rapide : puisqu'il n'est pas demandé de faire des modifications dans le code des services, nous allons utiliser la méthode **déclarative**.

Pour cela, nous devons [créer un yaml représentant la souscription](https://docs.dapr.io/developing-applications/building-blocks/pubsub/subscription-methods/) dans le dossier `src/Lab2/1-pubsub/components/`. Le nom du fichier n'a aucune importance.

Ce yaml prend cette forme:

```yaml
apiVersion: dapr.io/v1alpha1
# Le composant crée est une souscription...
kind: Subscription
metadata:
  # de nom sub-order.
  name: sub-order
# Cette souscription :
spec:
  # - utilise le composant dapr de nom "order-pub-sub"
  pubsubname: order-pub-sub
  # - est sur le topic "orders" (le même que celui dans lequel publie command-api)
  topic: orders
  # - redirige les messages vers le endpoint HTTP "/process-order"
  # qui est le endpoint ouvert sur le service "order-processing"
  route: /process-order
# Cette souscription ne s'applique que pour le service "order-processing"
scopes:
  - order-processing
```

{% endcollapsible %}

> **En pratique**: Implémenter le PUB/SUB entre `command-api` et `order-processing`

Une trace indiquant le succès devrait avoir cette forme

![expected result](/media/lab2/pubsub/expected-result.png)

**Note 1**: Dans un pattern Pub/Sub, un message est supprimé d'un _topic_ après que tous les consommateurs de ce topic l'ai reçu. Dans une véritable application E-commerce, où les microservices peuvent éventuellement _scaler_, c'est à dire augmenter leur nombre d'instances, les commandes pourraient être traitées plusieurs fois. La bonne approche dans ce genre de cas est d'utiliser une file de message, où le message serait supprimé après le premier traitement.
