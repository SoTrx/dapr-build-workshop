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
### In Application

> **Note**: The new version of the application is now located in `src/Lab2/3-bindings`

Let's revisit our main theme once again. This time, two new requirements:

1. We now need to interface with the information system of the supplier that replenishes our stock. The **stock-manager** service has a specific HTTP POST endpoint _/newproduct_.

   To simulate this, we can use a CRON job. If it's possible to use it directly in the application, we can use a specific Dapr binding for that.

2. The parent company of the enterprise has a dedicated mailing service. The **receipt-generator** service must be able to send emails to customers to confirm pre-orders. The mailing service is available at the following URL:

   ```shell
   # The "sig" parameter in the URL is intentionally incorrect; request the correct one on the workshop day
   https://prod-116.westeurope.logic.azure.com/workflows/0ceb8e48b2254276923acaf348229260/triggers/manual/paths/invoke?api-version=2016-10-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=lTON4ZTisB1iGA-6rJAlkoC8miHB9kyJp3No
   ```

   As both systems we need to interact with are external, we choose to use bindings.

The target setup is as follows:

![Step 3](/media/lab2/bindings/app-step-3-bindings.png)

> **Question**: Which binding should be used for the first requirement? What is the impact of the name of the HTTP POST endpoint for product reception (**newproduct**) on the binding?

**Solution**:
{%collapsible %}
Since we want to **react to an event triggered by an external system**, we need to use an _input binding_.

To simulate a CRON job, we can use the [associated binding](https://docs.dapr.io/reference/components-reference/supported-bindings/cron/).

As the HTTP endpoint is named newproduct, the `metadata.name` property of the binding should also be named newproduct.

{% endcollapsible %}

> **Question**: Which binding should be used for the second requirement?

**Solution**:
{%collapsible %}
Since we want to **send an event to an external system**, we need to use an _output binding_.

As the system used by the external service is a simple HTTP request, we can use the [HTTP binding](https://docs.dapr.io/reference/components-reference/supported-bindings/http/).

{% endcollapsible %}

> **In Practice**: Implement both bindings and verify their functionality. To check if the mailing service works, you can fill in the **MAIL_TO** environment variable of the **receipt-generator** service with a valid email. The sender of the email will be a Gmail address with the subject "Validated Command."

**Important**: The name of the binding should be `mail` because that is the one called in the code of **receipt-generator**.

A successful trace should look like this:

![Expected result](/media/lab2/bindings/expected-result.png)

**Solution**:
{%collapsible %}

##### Output Binding: mail

The output binding to use for the email is a simple HTTP binding. Create a new YAML file in the `src/Lab2/3-bindings/components` folder.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  # Important: As mentioned above, the binding name should be "mail"
  name: mail
spec:
  type: bindings.http
  version: v1
  metadata:
    # Use the invocation URL of the external service (be cautious with the API key)
    - name: url
      value: https://prod-116.westeurope.logic.azure.com/workflows/0ceb8e48b2254276923acaf348229260/triggers/manual/paths/invoke?api-version=2016-10-01&sp=%2Ftriggers%2Fmanual%2Frun&sv=1.0&sig=<api-key>
```

##### Input Binding: CRON

The input binding to use is a CRON. Create a new YAML file in the `src/Lab2/3-bindings/components` folder.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  # Important: As mentioned above, the binding name will correspond to
  # the method called on the services
  name: newproduct
spec:
  type: bindings.cron
  version: v1
  metadata:
    # The value of the CRON schedule is not crucial for this example
    - name: schedule
      value: "@every 15s"
```

{% endcollapsible %}

### Out of Curiosity: The "External System"

The presented "external" system is actually the following [LogicApp](https://docs.microsoft.com/fr-fr/azure/logic-apps/logic-apps-overview):

![Mailing](/media/lab2/bindings/logic-app-mailing.png)
