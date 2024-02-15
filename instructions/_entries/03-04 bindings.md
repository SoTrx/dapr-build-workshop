---
sectionid: lab2-bindings
sectionclass: h2
title: Bindings
parent-id: lab-2
---

### Overview

> **General Question**: What is a service-oriented architecture? What is an event-driven architecture? What is the difference between the two?

Solution:
{%collapsible %}
**/!\ Approximations /!\\**

A service-oriented architecture (SOA) is an architecture where a task to be accomplished is distributed among several programs (services) calling each other. Depending on the level of responsibility of each service, they can be referred to as microservices.

An event-driven architecture (EDA) is an architecture where communication between components of an application (which can be services) is ensured through events. These events typically transit through **event buses**.

Two significant differences between the two:

- **Coupling**
  - In SOA, services are more or less tightly coupled (URLs, message queues, etc.).
  - In EDA, the coupling is loose; those publishing events do not know who is listening, and vice versa.
- **Coherence**:
  - In SOA, when service A calls service B, the state of service A changes only after the success of the call (e.g., HTTP 200).
  - In EDA, when service A publishes an event, and service B listens to it, the state of service A has already changed at the time of publication since there is no return from service B.

We have seen two ways to approach communication with the last two exercises; now, there is external communication.

{% endcollapsible %}

### Dapr

Using the [documentation](https://docs.dapr.io/developing-applications/building-blocks/bindings/bindings-overview/), let's address these questions.

> **Question**: What is the purpose of a _binding_?

Solution:
{%collapsible %}
A binding is simply a way to interact with a system outside of our application's scope.

The principle is to bind a name to an external system and be able to call this name in the application's services.

The advantage is that this call is made transparently; the calling service does not know (and should not know) that the system called by the binding is external.

{% endcollapsible %}

> **Question**: What is the difference between an _input binding_ and an _output binding_? How is an _output binding_ different from a service invocation?

Solution:

{%collapsible %}
##### Input Binding

An _input binding_ allows reacting to a change of state in an external system.

An example would be reacting to a new message on a message queue located on another cloud provider.

##### Output Binding

An _output binding_ allows an external system to react to a change of state in our application.

An example would be defining a binding to a mail provider. Instead of having a dedicated service in the application, this binding could be called by all services that need it. The advantage is that if the mail provider changes, only the binding needs to be updated; the services remain unchanged.

#### Difference between Output Binding and Service Invocation

A service invocation is a **synchronous** invocation of an **internal** service. These services can be discovered by [discovery](https://docs.dapr.io/developing-applications/building-blocks/service-discovery/service-discovery-overview/). The call can be secured/authenticated automatically by using [Sentry](https://docs.dapr.io/concepts/dapr-services/sentry/).

An output binding is a **synchronous or asynchronous** invocation of an **external** service. Part of the security of bindings will necessarily be left to the external system.

{% endcollapsible %}

### En application

> **Note** : La nouvelle version de l'application se trouve désormais dans `src/Lab2/3-bindings`


Revenons une fois encore à notre fil rouge. Cette fois-ci, deux nouvelles demandes:

- Il faut maintenant pouvoir s'interfacer avec le système d'informations du fournisseur qui réapprovisionne notre stock. Le service **stock-manager** a un endpoint HTTP POST specifique _/newproduct_

Pour simuler ça, nous pouvons utiliser une tâche CRON. S'il est possible de l'utiliser directement dans l'application, nous pouvons utiliser un binding Dapr spécifique pour ça.

- La maison mère de l'entreprise dispose d'un service de mailing dedié. Le service **receipt-generator** doit être capable d'envoyer des mails aux clients pour confirmer les pré-commandes. Le service de mailing est disponible à l'URL suivante :

```shell
# Le paramètre "sig" de l'URL est volontairement faux, demandez la correction le jour du workshop
https://prod-116.westeurope.logic.azure.com/workflows/0ceb8e48b2254276923acaf348229260/triggers/manual/paths/invoke?api-version=2016-10-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=lTON4ZTisB1iGA-6rJAlkoC8miHB9kyJp3No
```

Comme les deux systèmes avec lesquels nous devons intéragir sont externes, nous choississons d'utiliser des bindings.

La cible est donc la suivante :

![Step 3](/media/lab2/bindings/app-step-3-bindings.png)

> **Question** : Quel binding utiliser pour le premier besoin ? Quelle est l'impact du nom du endpoint HTTP POST de reception de produit (**newproduct**) sur le binding ?

Solution :
{%collapsible %}
Etant donné que nous voulons **réagir à un événement lancé par un système externe**, nous devons utiliser un _input binding_.

Pour simuler une tâche CRON, nous pouvons utiliser le [binding associé](https://docs.dapr.io/reference/components-reference/supported-bindings/cron/)

Le endpoint HTTP s'appelant newproduct, la propriété `metadata.name` du binding devra également s'appeller newproduct.

{% endcollapsible %}

> **Question** : Quel binding utiliser pour le deuxième besoin ?

Solution :
{%collapsible %}
Etant donné que nous voulons **faire parvenir un événement à système externe**, nous devons utiliser un _output binding_.

Comme le système utilisé par le service externe est une simple requête HTTP, nous pouvons utiliser le [binding HTTP](https://docs.dapr.io/reference/components-reference/supported-bindings/http/)

{% endcollapsible %}

> **En pratique** : Mettez en place les deux bindings et vérifiez leur fonctionnement. Pour vérifier que le service de mailing fonctionne, vous pouvez remplir la variable d'environnement **MAIL_TO** du service **receipt-generator** avec un email valide. L'expediteur du mail sera une adresse gmail avec l'objet "Validated Command"

**Important**: Le nom du binding devra être `mail`, car c'est celui qui est appelé dans le code de **receipt-generator**

Une trace indiquant le succès devrait avoir cette forme :

![Expected result](/media/lab2/bindings/expected-result.png)

Solution :
{%collapsible %}

##### Output binding : mail

L'output binding à utiliser pour le mail est donc un simple binding http. Il suffit donc de créer un nouveau fichier yaml dans le dossier `src/Lab2/3-bindings/components`.

```yml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  # Important : Comme indiqué plus haut, le nom du binding doit être "mail"
  name: mail
spec:
  type: bindings.http
  version: v1
  metadata:
    # On utilise l'URL d'invocation du service externe (attention à la clef)
    - name: url
      value: https://prod-116.westeurope.logic.azure.com/workflows/0ceb8e48b2254276923acaf348229260/triggers/manual/paths/invoke?api-version=2016-10-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=<clef-api>
```

##### Input binding : CRON

L'input binding à utiliser est un CRON.
On crée donc encore une fois un nouveau fichier yaml dans le dossier `src/Lab2/3-bindings/components`.

```yml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  # Important : Comme indiqué ci-dessus, le nom du binding correspondra au
  # nom de la méthode appelée sur les services
  name: newproduct
spec:
  type: bindings.cron
  version: v1
  metadata:
    # La valeur du CRON n'a pas d'importance
    - name: schedule
      value: "@every 15s"
```

{% endcollapsible %}

### Par curiosité: Le "système externe"

Le système "externe" présenté est en fait la [LogicApp](https://docs.microsoft.com/fr-fr/azure/logic-apps/logic-apps-overview) suivante:

![Mailing](/media/lab2/bindings/logic-app-mailing.png)
