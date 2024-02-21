---
sectionid: lab2-service-invocation
sectionclass: h2
title: Invoke services
parent-id: lab-2
---

### Overview

> **General Question**: What is Service Meshing? Can you provide examples of software that offer this functionality?

Solution:
{% collapsible %}
The best way to define Service Meshing is to present the problem it addresses.

Let's take the example of two services, A and B. A wants to call B via HTTP.

To contact B, A will need information about its location. This information can take the form of a URL, an IP address, or even a pair (name, namespace) for Kubernetes, for example.

In any case, by providing this information to A, we couple A and B. Indeed, if B changes its location, A must also be updated to be functional.

This is a significant problem in the world of distributed systems, as services can change location quite often, whether in terms of nodes, namespaces, URLs, etc.

To address this issue, services like [_Istio_](https://istio.io/) or [_Open Service Mesh_](https://openservicemesh.io/) have emerged. The principle of these two software solutions is to maintain a service catalog associating the name with a location, adding an additional layer of abstraction. In this configuration, it is possible to call services by their name without revealing their location; _Istio_ or _Open Service Mesh_ are the only ones aware of it and handle the routing.

Applied to our example, a **greatly simplified** scenario could be as follows:

The _Istio_ instance could contain:

| Name | Location                 |
| --- | ----------------------- |
| A   | http://A.localapi.net   |
| B   | https://B.vendorapi.net |

A could then call B using only its name, with an address like `http://B.local`. _Istio_ would then route this call to the actual address of B, `https://B.vendorapi.net`.

This `http://B.local` address will remain valid even if the actual address of B changes, and only the _Istio_ instance needs to be updated.
{% endcollapsible %}

> **General Question**: What is gRPC? What are its advantages and disadvantages?

Solution:
{% collapsible %}
gRPC (Google Remote Procedure Call) is a framework created by Google that encapsulates remote procedure calls (RPC).

Unlike a REST API, which focuses on retrieving resources using a unified syntax, RPC is oriented towards **actions**. The simplest way to imagine RPC is to think of calling functions on a remote computer.

A unique feature of gRPC compared to other RPC frameworks is that it uses HTTP/2 and the [_Protobuf_](https://en.wikipedia.org/wiki/Protocol_Buffers) description language to define message contents.

The advantages of gRPC stem from this particularity, including increased speed and reduced size of sent content.

The main disadvantages are:

- gRPC is much more challenging for a human to interpret than REST, making it a solution more preferred for backend-to-backend service communication.
- gRPC has a longer development time.
{% endcollapsible %}

### Dapr

Using the [documentation](https://docs.dapr.io/developing-applications/building-blocks/service-invocation/service-invocation-overview/), let's address these questions.

> **Question**: What is the principle of service invocation? What is the path of a packet sent by service A invoking service B? What protocols are used during the different steps of a packet's journey from service A to service B?

Solution:
{% collapsible %}
The principle of service invocation is to invoke a method of a remote service securely and resiliently. Invoking a service also allows Dapr to automatically generate logs and traces.

A packet going from service A to service B would have the following path:

```sh
A ---HTTP/gRPC---> A's sidecar
#  URL localhost:3500/invoke/B/method/order

A's sidecar ---HTTP/gRPC---> DNS server
#  B A ? (requests the ipv4 address of B)

DNS server ---HTTP/gRPC---> A's sidecar
#  B A XXX.XXX.XX.XX

A's sidecar ---gRPC---> B's sidecar ---HTTP/gRPC---> B
#  Transmitting the call to the '/order' method of B

```

{% endcollapsible %}

> **Question**: In [the example on the documentation page](https://docs.dapr.io/developing-applications/building-blocks/service-invocation/service-invocation-overview#example), the URL allowing **pythonapp** to call **nodeapp** is _http://localhost:3500/v1.0/invoke/nodeapp/method/neworder_. Decompose this URL and explain its components.

Solution:

{%collapsible %}
Breaking down each component:

- **http://localhost:3500**: Call to the sidecar (unencrypted, unnecessary because of the same application sandbox)
- **v1.0**: Version of the Dapr API
- **invoke**: Use of the service invocation API
- **nodeapp**: Name of the service to call
- **method**: Calling a method on the service
- **neworder**: Name of the method to call on the service
{% endcollapsible %}

> **Question**: What is the difference between Service Meshing and Dapr's service invocation?

Solution:

{%collapsible %}
<u>On the Target</u>:
Service Meshing is deployed on infrastructure and is unique to that infrastructure. It is an OPS functionality.
Dapr's service invocation is independent of the infrastructure; it concerns DEV.

<u>On the Features</u>:
While both Service Meshing and Dapr's service invocation facilitate service-to-service calls, service meshing operates at the network level, whereas Dapr works at the application level. As a result:

- Dapr **adds** method discovery of the application in addition to resolving the service name.
- Dapr **does not enable** network redirections over the Internet (or a tunnel) in a multi-cloud application scenario, for example.

It is possible to use a service like _Istio_ in conjunction with Dapr, as the services do not have the same functional coverage.

See https://docs.dapr.io/concepts/service-mesh/
{% endcollapsible %}

> **Exploration**: In the page, _Dapr Sentry_ is mentioned. What is its role? Examine the `docker-compose.yml` file from the previous exercise. Is Sentry present? What do you deduce from this?

Solution:

{%collapsible %}
Sentry enables **encryption** and **mutual authentication** of communications between services. It allows mTLS communication between services, acting as a storage/broker of certificates.

Sentry is a completely optional service. If it is not present at the start of the sidecars (and its address specified in the startup command), communications will simply not be encrypted.

Dapr, therefore, has a modular architecture, and there are other optional services:

- **[Placement](https://docs.dapr.io/concepts/dapr-services/placement/)**, enabling the use of the [Actor model](https://en.wikipedia.org/wiki/Actor_model)
- The internal Dapr **[Name Resolution Component](https://docs.dapr.io/reference/components-reference/supported-name-resolution/)** is also modular. By default, flat resolution ([mDNS](https://en.wikipedia.org/wiki/Multicast_DNS)) is used, but **coreDNS

{% endcollapsible %}

## In Practice:

> **Note**: The new version of the application is now located in `src/Lab2/2-service-invocation`.

It's time to pick up the main thread again. Always aiming to make our pre-order application complete, two new services have been added:

- **stock-manager** (in Go): Once an order is validated, **order-processing** calls the _/stock_ method of **stock-manager** to add the order to the required stocks.
- **receipt-generator** (in Rust): Once an order is validated, **order-processing** calls the _/_ method of **receipt-generator** to generate a confirmation.

The name of each service is also its app-id.

The new target is as follows:

![Expected result](/media/lab2/service-invocation/step-2-service-invocation.png)

> **Question**: What is the URL that **order-processing** should use to call **stock-manager**? Explain.

**Hint**: The Dapr service invocation API is available [here](https://docs.dapr.io/reference/api/service_invocation_api/).

Solution:
{% collapsible %}
According to the documentation, the URL for a service invocation is of the form:

```sh
PATCH/POST/GET/PUT/DELETE http://localhost:3500/v1.0/invoke/<appId>/method/<method-name>
```

where:

- **localhost:3500** is the address of the sidecar
- **invoke** is the prefix of the invocation API
- **\<appId\>** is the service id to call, as declared by the `--app-id` option of the Dapr command line
- **\<method-name\>** is the name of the method to call on the remote service

In this specific case, the service to call is **stock-manager**, specifically the _/stock_ method.

The desired URL is, therefore:

```sh
http://localhost:3500/v1.0/invoke/stock-manager/method/stock
```

{% endcollapsible %}

> **Question**: What is the URL that **order-processing** should use to call **receipt-generator**? Explain.

Solution:
{% collapsible %}
According to the documentation, the URL for a service invocation is of the form:

```sh
PATCH/POST/GET/PUT/DELETE http://localhost:3500/v1.0/invoke/<appId>/method/<method-name>
```

where:

- **localhost:3500** is the address of the sidecar
- **invoke** is the prefix of the invocation API
- **\<appId\>** is the service id to call, as declared by the `--app-id` option of the Dapr command line
- **\<method-name\>** is the name of the method to call on the remote service

In this specific case, the service to call is **receipt-generator**, specifically the _/_ method.

The desired URL is, therefore:

```sh
http://localhost:3500/v1.0/invoke/receipt-generator/method/
```

{% endcollapsible %}

> **In Practice**: Using the answers to the two previous questions, fill in the environment variables **RECEIPT_GENERATOR_INVOKE_URL** and **STOCK_MANAGER_INVOKE_URL** in `docker-compose.yml`. Execute the docker-compose file and place a pre-order via the web interface (localhost:8089).

**Reminder**: To run a docker-compose file, use the following command:

```sh
docker compose rm -fsv ; docker compose up --no-attach redis
```

The success trace should look like this:

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

**Note**: It is also possible to limit which service(s) can call a service. For this, a configuration object exists that can be passed as an argument to each sidecar using the CLI switch **--config**. Here is an example where only **order-processing** would have the right to call the microservice.

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

To apply this configuration to **receipt-generator**, for example, you would need to:

- Create a config folder and save the above configuration in it (with the name, for example, `config.yml`).
- Modify the docker-compose file as follows:

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