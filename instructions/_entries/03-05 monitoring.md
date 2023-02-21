---
sectionid: lab2-monitoring
sectionclass: h2
title: Observabilité
parent-id: lab-2
---

### Observer les chaînes d'appels

Il est possible d'utiliser la solution Open-source [Zipkin](https://zipkin.io/) pour tracer les appels entre les services. Il suffit pour ça de configurer Dapr de la manière suivante

```yml
# Fichier : src/Lab2/4-observability/config/config-tracing.yml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprConfig
  namespace: default
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      # zipkin:9411 est accessible grâce à docker compose. Il faudrait un service dedié
      # dans un environnement Kubernetes
      endpointAddress: "http://zipkin:9411/api/v2/spans"
```

> **Question**: Comparez le déploiement de l'exercice précédent (`src/Lab2/3-bindings/docker-compose.yml`) et de l'exercice en cours (`src/Lab2/4-observability/docker-compose.yml`). Comment configurer Dapr pour prendre en compte Zipkin ?

Solution:
{% collapsible %}

En prenant comme exemple le déploiement du service **command-api**.

```diff
  ############################
  # Command API
  ############################
  command-api:
    image: dockerutils/command-api
    environment:
     - PUB_URL=http://localhost:3500/v1.0/publish/order-pub-sub/orders
  command-api-dapr:
    image: "daprio/daprd:edge"
    command: ["./daprd",
     "-app-id", "command-api",
     "-app-port", "80",
+    "-config", "/config/tracing-config.yml",
     "-components-path", "/components"]
    volumes:
        - "./components/:/components"
+       - "./config/:/config"
    network_mode: "service:command-api"
```

En plus du déploiement de Zipkin en lui-même, la configuration de Dapr est appliquée à l'aide de l'argument **-config**. Le fichier de configuration en lui-même est monté dans un volume différent de celui des composants.

Dans un contexte Kubernetes, la configuration serait appliquée à l'aide d'un `kubectl apply -f config-tracing.yml`
{% endcollapsible %}

Déployez `src/Lab2/4-observability/docker-compose.yml` à l'aide de la commande suivante
et passez quelques commandes via l'interface. Naviguez à l'adresse `localhost:9415` pour vous rendre sur l'interface de Zipkin. Dans l'onglet "dépendances", prenez une plage large (ex: [j-1, j+1]) et observez le diagramme.

> **Question** : Quels sont les services affichés ? Comment ces services communiquent-ils ?

Solution:
{% collapsible %}
Les services affichés sont :

- **command-api**
- **order-processing**
- **receipt-generator**
- **stock-manager**

Communications :

- **command-api** --> **order-processing** : pub/sub
- **order-processing** --> **receipt-generator** : invocation de service
- **order-processing** --> **stock-manager** : invocation de service

{% endcollapsible %}

> **Question** : Quels sont le(s) service(s) manquant(s) ? Pourquoi ?

**Indication** : Il peut y avoir un certain temps d'attente avant que le diagramme s'affiche. Vous pouvez utiliser l'image ci-dessous si le temps manque.

{% collapsible %}
![zipkin deps](/media/lab2/observability/zipkin-deps.png)
{% endcollapsible %}

Solution:
{% collapsible %}
Le service manquant est **command-frontend**. C'est en effet le seul service qui ne possède pas de sidecar, étant donné qu'il se trouve côté client.

On remarquera aussi que les bindings ne sont pas représentés. En effet ceux-ci font le lien entre notre application et des systèmes externes, qui ne sont donc pas retracés dans un compte rendu sur les chaînes d'appels internes.
{% endcollapsible %}

### Observer les métriques

Un autre axe de l'observabilité est celui des "métriques" des conteneurs. Ces métriques sont des statistiques généralistes qui permettent d'avoir une vue opérationelle sur le cluster.

Ces statistiques contiennent notamment :

- Utilisation CPU des sidecars
- Utilisation RAM des sidecars
- Latence moyenne entre chaque service et son sidecar
- Uptime des sidecars
- Etats des composants
- Statistiques des appels HTTP/gRPC

La liste complète des métriques envoyées par chaque service de Dapr est disponible [ici](https://github.com/dapr/dapr/blob/master/docs/development/dapr-metrics.md)

Toutes ces métriques sont émises dans [un format ouvert](https://github.com/prometheus/docs/blob/main/content/docs/instrumenting/exposition_formats.md) et peuvent être récupérées et analysées par des outils dédiées tels que [Azure Monitor](https://azure.microsoft.com/fr-fr/services/monitor/#overview), ou [Prometheus](https://prometheus.io).

[Chaque release](https://github.com/dapr/dapr/releases) de Dapr propose également des dashboards [Grafana](https://grafana.com/) préconçus permettants d'avoir un récapitulatif graphique des métriques collectées.

![grafana dapr doc](/media/lab2/metrics/grafana-doc-example.png)

Un guide est disponible [sur la documentation](https://docs.dapr.io/operations/monitoring/metrics/grafana/) expliquant la mise en place de ces dashboards.

#### En application (Facultatif)

> **Important** : Cette section se concentrera sur l'obtention de certaines métriques sur l'exemple fil rouge. Il faut cependant noter que les dashboards grafana fournis par l'équipe Dapr sont conçus pour Kubernetes, certaines métriques ne seront pas disponibles.

Pour obtenir les métriques de notre application, nous allons donc avoir besoin de rajouter deux services à notre déploiement : Prometheus et Grafana.

Prometheus est un outil d'analyse de séries temporelles (~= variables évoluants sur un axe de temps). Il permet de récupérer et d'aggréger des informations de *n* sources HTTP.

Grafana est un outil de visualisation souvent utilisé en conjonction de Prometheus. Il est capable de créer des _dashboards_ à partir de plusieurs sources de données, offrant alors une visualisation fine des données à l'utilisateur.

> **En pratique**: Déployez l'application spécifiée par `src/Lab2/5-metrics/docker-compose.yml`. Naviguez maintenant vers Grafana à l'adresse **localhost:9417**, puis choississez le _dashboard_ "dapr-sidecar-dashboard" dans _Dashboards -> Browse_. Qu'observez-vous ?

**Indication** : Il vous faudra sûrement réduire la fenêtre d'observation à 5 minutes pour voir une évolution dans les graphiques, ainsi que de lancer quelques commandes de Charentaises

Solution:
{% collapsible %}
![sample grafana result](/media/lab2/metrics/sample-grafana-result.png)

On observe que les métriques présentes sont celles des latences et des composants.

L'utilisation CPU/RAM n'est elle pas renseignée pour cet exemple. En effet, ces données sont extrapolées à partir des [métriques Kubernetes](https://github.com/kubernetes/kubernetes/blob/master/test/instrumentation/testdata/stable-metrics-list.yaml), qui ne sont pas présentes dans un contexte docker compose

{% endcollapsible %}

##### Par curiosité : Quelques détails sur Prometheus et docker compose

Le déploiement de cette section n'est pas expliqué à proprement parler, seulement exécuté. La raison à cela est que faire fonctionner Prometheus avec Dapr sur docker compose demande d'utiliser une configuration particulière qui n'a pas sa place dans un scénario de production.

En effet, les métriques de Dapr sont émises par chaque sidecar sur leur port respectif 9090 (par défaut).
Dans le contexte d'une utilisation sur Kubernetes ou sans orchestrateur, chaque sidecar émettrait sur le même endpoint sur le port 9090. Il n'y aurait plus qu'alors qu'à renseigner ce endpoint dans Prometheus.

Cependant, la gestion des [docker networks](https://docs.docker.com/network/) dans docker compose ne permet pas à chaque sidecar d'émettre sur le même endpoint. Afin de retrouver le comportement voulu, il faut alors lister explicitement chacun des services dans la configuration de Prometheus.

```yml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: "dapr"
    scrape_interval: 5s

    static_configs:
      - targets:
          [
            "command-api:9090",
            "order-processing:9090",
            "stock-manager:9090",
            "receipt-generator:9090",
          ]
```

De plus, chaque couple (service, sidecar) partage une même interface réseau,  il faut donc de manière assez contre-intuitive exposer le port 9090 du service pour atteindre ce même port sur le sidecar.

```diff
  order-processing:
    image: dockerutils/order-processing
+    expose:
+      - 9090
      ...
  order-processing-dapr:
    image: "daprio/daprd:edge"
      ...
    network_mode: "service:order-processing"
```

Une fois Prometheus configuré, Graphana lui ne pose pas de problème particulier.

```ini
[auth]
# Remove the login prompt
disable_login_form = true

[auth.anonymous]
# enable anonymous access
enabled = true
# And give "anonymous" admin privileges
org_role = Admin
```

### Observer les logs

Le dernier axe de l'observabilité que nous allons aborder est celui des logs. La possibilité de stocker et d'analyser les logs est une partie intégrante de la vie d'une application distribuée – peut être même encore plus que les parties précédentes – et il est courant que chaque développeur ait déjà une solution plus ou moins managée avec laquelle il est familier.

Il n'est donc pas question ici de discuter de la manière dont les logs des services en eux-mêmes sont traités, cette partie va plutôt se concentrer seulement sur **les logs des sidecars**.

Le support utilisé pour cet exemple sera une pile ELK (Elasticsearch, Logstash, Kibana). Ce n'est cependant pas la seule solution supportée par Dapr.

> **En pratique**: Déployez l'application spécifiée par `src/Lab2/6-logs/docker-compose.yml`. Naviguez maintenant vers Kibana à l'adresse **localhost:5601**

**Note**: Il est possible que le déploiement échoue avec une erreur de type _Dial [1]::24224_, relancez simplement la commande dans ce cas

L'instance de Kibana est déjà configurée pour recevoir les logs des sidecars (voir la partie [**Détails**](#détails) ci-dessous). Il faut maintenant configurer cette instance pour les analyser

Pour cela:

- Une fois sur l'interface, choississez l'option _Explore on my own_ quand il sera proposé d'ajouter des intégrations
- Cliquez en haut à gauche sur le menu (**☰**), puis sur _Stack Management_ dans la section _Management_
- Une fois sur la nouvelle page, dans le panneau de gauche, cliquez sur _Data View_ dans la section _Kibana_
- Cliquez sur _Create Data View_
- Il sera demandé de renseigner un index. Il ne devrait n'y en avoir qu'un seul disponible de la forme **fluent-\<\>**. Renseignez "fluent\*" dans le champ de gauche
- Cliquez sur _Create Data View_

En créant cette vue, vous aurez la liste des variables comprises dans les logs.

> **Question**: En comparant les variables affichées avec [le format de logs de Dapr](https://docs.dapr.io/operations/monitoring/logging/logs/#log-schema), quelles différences remarquez-vous ? Pourquoi ?

Solution :

{% collapsible %}
![Dapr log items list](/media/lab2/logs/log-items-full.png)

Par rapport au format de Dapr, il y a de nombreuses variables supplémentaires. Ces variables proviennent du [format ECS](https://www.elastic.co/guide/en/observability/8.3/logs-app-fields.html) utilisé.  
{% endcollapsible %}

Il suffit maintenant pour consulter les logs de cliquer à nouveau sur **☰** puis de cliquer sur _Discover_ dans la section _Analytics_.

**Remarque** : On remarque des doublons dans les attributs, suffixés par ".keyword". Il s'aggit d'une spécificité de Elasticsearch : Lors de la rencontre d'une chaîne de charactères, Elasticsearch va l'indexer à la fois en tant que type _TEXT_, champ dans lequel il est possible de rechercher un sous-texte, et en tant que type _KEYWORD_, non indexé. Il est cependant possible de spécifier quel comportement adopter pour chacun des champs.

> **En pratique** : Isolez les logs du conteneur **order-processing**. Commentez les attributs

{% collapsible %}

Pour isoler les logs du conteneur **order-processing**, il suffit de chercher "app-id : order-processing" dans la barre de recherche.

Une ligne de log a cette forme

```jsonc
// Note : Les attributs .keyword sont ignorés
{
  "_index": "fluentd-20220704",
  "_id": "sepuyYEB3S4ZspuilsDO",
  "_version": 1,
  "_score": null,
  "fields": {
    // Payload de Dapr. On remarque que l'on retrouve chacun de ces attributs
    // séparément dans l'objet "fields". C'est un effet de la configuration de FluentD
    // qui parse le JSON de cet attribut et l'intègre à l'objet parent
    "log": [
      "{\"app_id\":\"order-processing\",\"instance\":\"d900866e4786\",\"level\":\"info\",\"msg\":\"application configuration loaded\",\"scope\":\"dapr.runtime\",\"time\":\"2022-07-04T13:37:51.345159194Z\",\"type\":\"log\",\"ver\":\"edge\"}"
    ],
    // Payload -> Descripteur de fichier d'origine
    "source": ["stdout"],
    // Payload -> Message de log à proprement parler
    "msg": ["application configuration loaded"],
    // Payload -> Log type
    "type": ["log"],
    "scope": ["dapr.runtime"],
    // Payload -> APP-id du service ayant emis le log
    "app_id": ["order-processing"],
    // Payload ->Version de Dapr
    "ver": ["edge"],
    // Payload ->Log level
    "level": ["info"],
    // Meta -> Temps de reception de la ligne de log
    "@timestamp": ["2022-07-04T13:37:51.345Z"],
    // Meta -> Nom du conteneur vu par docker compose
    "container_name": ["/6-logs_order-processing-dapr_1"],
    // Meta -> UUID du conteneur docker
    "container_id": [
      "1f8457fe7a405ecf8443304558aedbeaa4bd9eff0a45ecd2b7aa24caf4879e73"
    ]
  },
  // Date de réception en format timestamp pour trier les logs
  "sort": [1656941871345]
}
```

{% endcollapsible %}

#### Détails

La manière dont les logs sont envoyés vers Kibana est d'utiliser [FluentD](https://docs.fluentd.org/) en tant que [logging driver](https://docs.docker.com/config/containers/logging/configure/).

Dapr est ensuite configuré pour afficher les logs en format JSON sur stdout avec l'option de commande `-log-as-json`.

```diff
  command-api-dapr:
    image: "daprio/daprd:edge"
    command: ["./daprd",
     "-app-id", "command-api",
     "-app-port", "80",
+      "-log-as-json", "true",
     "-config", "/config/tracing-config.yml",
     "-components-path", "/components"]
     ...
+   logging:
+     driver: "fluentd"
+      options:
+       fluentd-address: localhost:24224
+       tag: httpd.access
```

Cette méthode étant propre à l'orchestrateur docker compose, elle n'est pas détaillée dans le coeur de l'activité.

Sur Kubernetes le déploiement serait toutefois similaire. Il suffirait en effet de déployer FluentD sur le cluster et d'ajouter l'annotation `log-as-json` au déploiement des services pour obtenir le même resultat. Un tutoriel complet est disponible [sur le site de Dapr](https://docs.dapr.io/operations/monitoring/logging/fluentd/).
