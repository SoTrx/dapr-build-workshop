---
sectionid: lab1-enterdapr
sectionclass: h2
title: Enter Dapr
parent-id: lab-1
---

In looking at the code of the Node and Python services, answer the following questions:

> **Question**: How do the three services communicate with each other?

Solution:
{% collapsible %}
The three services communicate **directly**:

- The Python service calls the Node service via an **HTTP request** to its server's URL.
- The Node service calls Redis using the **[ioredis](https://www.npmjs.com/package/ioredis)** library, which encapsulates the **RESP** protocol, Redis's specific protocol.
  {% endcollapsible %}

The role of Dapr is to enable a kind of architectural decoupling. Instead of having to link the components together – using specific libraries, for example – we can add an additional level of abstraction to simplify the code.

More importantly, this decoupling allows the developer of a service to **offload the responsibility of implementation**.

To illustrate this, we will deploy another version of the presented application that uses Dapr.

> **Note**: This new application is located in the `src/Lab1/1-decoupling/withDapr` folder.

This new version can be executed using the following command:

```shell
# Ignore logs from the mongo and redis services to avoid log pollution
docker compose up --no-attach mongo --no-attach redis
```

> **Question**: Compare the two versions (with and without Dapr) of the NodeJS service. How does the service retrieve the message?

Solution:
{% collapsible %}
A generic HTTP call is made to the port 3500 **of the localhost interface** of the service. This port is the default one used by Dapr.

```ts
// src/Lab1/1-decoupling/withDapr/node/src/app.ts, line 9
const daprPort = env.DAPR_HTTP_PORT || 3500;
const stateStoreName = `statestore`;
const stateUrl = `http://localhost:${daprPort}/v1.0/state/${stateStoreName}`;
```

Indeed, Dapr works with the **sidecar** principle. A sidecar is a small program that attaches to each of the services we deploy. Each pair (main application, sidecar) shares the same localhost interface.

Whenever the main application wants to communicate with another service, it can simply call its sidecar, which will, in turn, handle the call.

{% endcollapsible %}

> **Question**: In the `docker-compose.yml` file, where are the Dapr *sidecars*? What is the docker-compose instruction that allows each *sidecar* to access the localhost interface of the service it is attached to? Does the Redis container have a sidecar?

Solution:
{% collapsible %}

```yml
  # Main application
  nodeapp:
    build: ./node
  # Sidecar
  nodeapp-dapr:
    image: "daprio/daprd:edge"
    # Sidecar configuration for the node application
    command: ["./daprd",
    # Application ID for Dapr
    # This ID allows later service invocation (see Lab2)
     "-app-id", "nodeapp",
    # Main application listening port
    # Used when the sidecar transmits information to the main application
     "-app-port", "3000",
     # Verbosity level of the sidecar
     # "info" or "debug" will provide more details on the sidecar's operation
     "-log-level", "warn",
     # Where to find Dapr components (next question)
     "-components-path", "/components"]
    volumes:
        - "./components/:/components:ro"
    # Instructs the sidecar to share its localhost interface with the main application
    network_mode: "service:nodeapp"
```

The Redis container does not have a sidecar since it is not a service but only a means of communication (or storage support). It does not need the features provided by Dapr.

{% endcollapsible %}

> **Question**: Still in the `docker-compose.yml` file, we can notice that a [volume](https://docs.docker.com/storage/volumes/) named **components** is mounted on each of the sidecars. What does the folder contain? What could be the purpose of the files it contains?

Solution:
{% collapsible %}

This folder contains a YAML file that looks like this:

```yml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  # Among the state management components, we select Redis
  type: state.redis
  version: v1
  # Provide the address and connection credentials to Redis
  metadata:
    - name: redisHost
      # We can resolve the Redis IP by its name
      # thanks to the Docker-compose hello-dapr network
      value: redis:6379
    - name: redisPassword
      value: ""
```

This is a definition of a Dapr **[component](https://docs.dapr.io/concepts/components-concept/)**.

These components are a central idea of Dapr, and they are what allows this notion of decoupling.
Here, we declaratively announce that we want to use Redis as a state storage component.
With this declaration, all calls that applications make to retrieve or store a state will be
redirected to Redis. Changing this component would change the underlying storage support.

{% endcollapsible %}

With all this information in mind, we can recap.

![First app with Dapr](/media/lab1/first-app-dapr.png)

The path taken by our state is as follows:

- The Python service communicates the state to its sidecar with a URL of the form:

```bash
# The URL contains the ID specified in the Node service's sidecar.
# This URL is intended to invoke the "neworder" method on the "nodeapp" service.
# We will see this in detail in the second lab.
http://localhost:3500/v1.0/invoke/nodeapp/method/neworder
```

- The Python service's sidecar communicates the state to the Node service's sidecar, where `-l'app-id` is the one specified in the URL, here **nodeapp**.
- The Node service's sidecar forwards the state to the Node service.
- The Node service tells its sidecar that it wants to store a state in the storage component named **storename** with a call like:

```shell
POST /state/storename
```

- Dapr then looks at its components and retrieves the storage management component named **storename**. This component specifies the host and connection parameters of the component's implementation, here Redis. The call is then forwarded to Redis.

> **Practical Exercise**: Using [the dedicated documentation](https://docs.dapr.io/reference/components-reference/supported-state-stores/setup-mongodb/), migrate the Redis state manager to MongoDB.

**Hint**: The MongoDB database is already present in the `docker-compose.yml` file; you just need to use it.

Solution:
{% collapsible %}
To change the state manager from Redis to MongoDB, you simply need to change the dedicated component, `src/Lab1/1-decoupling/withDapr/components/statestore.yaml`.

Currently, the component uses Redis:

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

Simply changing the component type in MongoDB and access credentials allows you to switch the state manager:

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
  