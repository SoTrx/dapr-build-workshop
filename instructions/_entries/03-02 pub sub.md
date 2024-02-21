---
sectionid: lab2-pubsub
sectionclass: h2
title: Pub Sub
parent-id: lab-2
---


### Overview

> **General Question**: What is the _Publish Subscribe (PUB/SUB)_ pattern? What need does it address?

Solution:
{% collapsible %}
The publish/subscribe pattern is a form of asynchronous communication.

In this mode of communication, there are two types of actors:

- **Producers** (_Publishers_), who generate content, often in the form of messages.
- **Consumers** (_Subscribers_), who await content.

By analogy, if direct invocation could be compared to a phone call, pub/sub would be more like an email inbox that each participant can check at their convenience.

Most of the time, pub/sub operates in a **_one-to-many_** configuration, meaning a message from a producer reaches all consumers. However, there are **_one-to-one_** configurations where a message from a producer is received by only one consumer.

{% endcollapsible %}

### Dapr

Using the [documentation](https://docs.dapr.io/developing-applications/building-blocks/pubsub/), let's address these questions:

> **Question**: How does Dapr's PUB/SUB feature work? What is the path of a message from its sending service A to service B?

Solution:
{% collapsible %}
![Pub/Sub overview](/media/lab2/pubsub/pubsub-overview.png)

Referring to the documentation image:

- The message passes from the service to its sidecar.
- The sidecar resolves the component and redirects the message to the underlying implementation.
- The implementation notifies all sidecars with the message content.
- Sidecars redirect the message content to an HTTP route in their respective services.
{% endcollapsible %}

> **Question**: What is the delivery guarantee associated with PUB/SUB functionality? What are the advantages and disadvantages?

Solution:
{% collapsible %}
The guarantee is **at least once** ([Link in the documentation](https://docs.dapr.io/developing-applications/building-blocks/pubsub/pubsub-overview/#at-least-once-guarantee)). This guarantee means that each message sent by a producer will be received at least once by each consumer. The advantage is that it prevents message loss, at least as long as at least one consumer is subscribed at the time of message sending.

The main disadvantage is the possibility of receiving the same message multiple times. This drawback can be addressed at the application level. For example, by keeping track of the IDs of the last X processed messages or by designing a service to be resistant to duplication.

#### Going Further

In designing a communication system, if the possibility of duplication must be absolutely avoided, one should lean towards an **at most once** guarantee. In this case, a message will be delivered at most once, avoiding duplication but risking message loss.

The ideal guarantee, **exactly once**, is not generally offered. There will always be edge cases where a transaction needs to be retried (e.g., a consumer failing during message reception).

{% endcollapsible %}

> **Question**: There are two methods to subscribe to a _topic_ in Dapr, what are they? In what cases should both be used?

Solution:
{% collapsible %}

To subscribe to a _topic_, Dapr offers two methods:

##### In Code

This is the "classic" way. The SDK is used to define a callback when a message is received.

```ts
import { DaprServer } from "@dapr/dapr";
const server = new DaprServer();
// Subscribe to the "orders" topic on the "pubsub" component
await server.pubsub.subscribe("pubsub", "orders", async (orderId) => {
  console.log(`Subscriber received: ${JSON.stringify(orderId)}`);
});
await server.startServer();
```

##### Declaratively

The other way is to declare the subscription as you would with a component.

```yaml
apiVersion: dapr.io/v1alpha1
kind: Subscription
metadata:
  name: order
spec:
  # On the "pubsub" component...
  pubsubname: pubsub
  # when a message is received on the "orders" topic...
  topic: orders
  # send the message content to the /checkout endpoint...
  route: /checkout
# for the "orderprocessing" and "checkout" services
scopes:
  - orderprocessing
  - checkout
```

The advantage is that the "orderprocessing" and "checkout" services can remain generic. They just need to use an HTTP server listening on the /checkout endpoint to receive the message.

{% endcollapsible %}

> **Extension**: The documentation highlights a feature called Content-Based Routing. What is the principle of this feature? What need does it address?

{% collapsible %}
Content-Based Routing allows Dapr to choose the HTTP endpoint to which a message will be redirected based on the content of the message.

```yaml
apiVersion: dapr.io/v2alpha1
kind: Subscription
metadata:
  name: myevent-subscription
spec:
  # On the "pubsub" component...
  pubsubname: pubsub
  # when a message is received on the "inventory" topic...
  topic: inventory
  # send the message content...
  routes:
    rules:
      # to the "/widgets" endpoint if the message type is "widget"...
      - match: event.type == "widget"
        path: /widgets
      # to the "/gadgets" endpoint if the message type is "gadget"...
      - match: event.type == "gadget"
        path: /gadgets
    # otherwise, send it to the "/products" endpoint
    default: /products
# for the "app1" and "app2" applications
scopes:
  - app1
  - app2
```

This feature is useful in cases where the application receives a large number of different events. It helps to:

- Avoid creating a large number of topics, one for each edge case of the application. On public clouds, this resolves a cost issue.
- Prevent the application itself from handling routing, which would introduce unnecessary complexity.

However, excessive use of this feature could make it challenging to understand the application's flow. The best way to avoid such a situation is to plan its use during the design phase.

{% endcollapsible %}

### In Application

Here is the current state of the pre-order application:

![Step 0](/media/lab2/app-step-0.png)

The frontend serves as the showcase of our site, running on the client's browser (which is why it doesn't have a Dapr sidecar attached), and the API is the entry point to the backend architecture.

> **Note**: This new application is located in the `src/Lab2/1-pubsub` folder.

To start the application, simply run the command:

```shell
# docker compose rm ensures that changes in docker-compose.yml are applied
docker compose rm -fsv ; docker compose up --no-attach redis
```

The frontend should be available at `localhost:8089`. Clicking the **Order** button triggers a command.

The next step in developing this application is to add an `order-processing` service that will receive and process orders.

To avoid overwhelming the service in case of high demand, we will opt for asynchronous order processing, using the Pub/Sub pattern (not necessarily ideal, see Note 1).

Here's the target:

![Step 1](/media/lab2/pubsub/app-step-1-pubsub.png)

Note:

- Each text in purple is an environment variable to be filled in `src/Lab2/1-pubsub/docker-compose.yml`.
  - **PUB_URL**: URL called by the **command-api** service to publish a message. This is a call to its sidecar, so it will be prefixed by _http://localhost:3500_.
  - **COMMAND_API_HOST**: URL of the **command-api** service with respect to **command-frontend**. This value is already pre-filled.
- **/process-order**: HTTP POST endpoint for message processing. New messages should be redirected to this URL.

> **Question**: In the `src/Lab2/1-pub-sub/components` directory, there is now a new file `pubsub.yaml`. What is its purpose? What is the name of the associated **Dapr component**?

Solution:

{% collapsible %}

This new file is defined as follows:

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

This is the Dapr component that connects to the Pub/Sub implementation, here Redis.

The component's name (the value of the yaml key **metadata.name**) is `order-pub-sub`. It's important not to confuse this name with the file name, which is simply `pubsub`.

{% endcollapsible %}

> **Question**: Using the above diagram, identify how to implement PUB/SUB for `command-api` and `order-processing`.

**Hint**: The URL that allows **command-api** to publish a message is a call to its sidecar. Dapr's pub/sub API is detailed [here](https://docs.dapr.io/reference/api/pubsub_api/).

**Hint 2**: **No modifications** are required in the services' code! Remember that there are several ways to subscribe to a *topic*.

Solution:

{% collapsible %}

Among these two services, **command-api** is the producer. As SDKs are not used, the correct call to its sidecar must be found.

##### Command API

The **command-api** service is the producer. To publish an event, it needs the URL to which it publishes its content.

This URL can be found in the [documentation](https://docs.dapr.io/reference/api/pubsub_api/), and it is in the form:

```sh
POST http://localhost:<daprPort>/v1.0/publish/<pubsubname>/<topic>
```

where:

- localhost:3500 is the address of the sidecar.
- publish is the prefix of the pub/sub API.
- <pubsubname> is the name of the pub/sub component to use, here **order-pub-sub**.
- <topic> is the topic to publish the message to. This can be any value. Here, we will use **orders**.

Once the variables are filled, the invocation URL is:

```sh
http://localhost:3500/v1.0/publish/order-pub-sub/orders
```

Now, simply fill in this URL in the environment variables of the `src/Lab2/1-pubsub/docker-compose.yml` file:

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

##### Order Processing

**Order-processing** is the consumer. It needs to subscribe to the **orders** topic, which is the one **command-api** uses for this example.

Based on the previous questions, we know there are two methods to subscribe to a topic:

- In code
- Declaratively

The choice here is quick: since no modifications are required in the services' code, we will use the **declarative** method. To do this, we need to [create a yaml representing the subscription](https://docs.dapr.io/developing-applications/building-blocks/pubsub/subscription-methods/) in the `src/Lab2/1-pubsub/components/` directory. The filename does not matter.

This yaml takes this form:

```yaml
apiVersion: dapr.io/v1alpha1
# The created component is a subscription...
kind: Subscription
metadata:
  # with the name sub-order.
  name: sub-order
# This subscription:
spec:
  # - uses the Dapr component named "order-pub-sub"
  pubsubname: order-pub-sub
  # - is on the "orders" topic (the same one that command-api publishes to in this example)
  topic: orders
  # - redirects messages to the HTTP endpoint "/process-order"
  # which is the open endpoint on the "order-processing" service
  route: /process-order
# This subscription applies only to the "order-processing" service
scopes:
  - order-processing
```

{% endcollapsible %}

> **In Practice**: Implement PUB/SUB between `command-api` and `order-processing`.

A successful trace should look like this:

![expected result](/media/lab2/pubsub/expected-result.png)

**Note 1**: In a Pub/Sub pattern, a message is removed from a _topic_ after all consumers of that topic have received it. In a real e-commerce application, where microservices may potentially scale, meaning increase their number of instances, orders could be processed multiple times. The proper approach in such cases is to use a message queue, where the message would be removed after the first processing.
