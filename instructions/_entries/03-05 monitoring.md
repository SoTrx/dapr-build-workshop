---
sectionid: lab2-monitoring
sectionclass: h2
title: Observability
parent-id: lab-2
---
### Observing Call Chains

It is possible to use the Open-source solution [Zipkin](https://zipkin.io/) to trace calls between services. To achieve this, configure Dapr as follows:

```yml
# File: src/Lab2/4-observability/config/config-tracing.yml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: daprConfig
  namespace: default
spec:
  tracing:
    samplingRate: "1"
    zipkin:
      # zipkin:9411 is accessible through docker compose. A dedicated service
      # would be required in a Kubernetes environment
      endpointAddress: "http://zipkin:9411/api/v2/spans"
```

> **Question**: Compare the deployment of the previous exercise (`src/Lab2/3-bindings/docker-compose.yml`) and the current exercise (`src/Lab2/4-observability/docker-compose.yml`). How do you configure Dapr to take Zipkin into account?

Solution:
{% collapsible %}

Taking the deployment of the **command-api** service as an example:

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

In addition to deploying Zipkin itself, Dapr configuration is applied using the **-config** argument. The configuration file itself is mounted in a volume separate from the components.

In a Kubernetes context, the configuration would be applied using `kubectl apply -f config-tracing.yml`.
{% endcollapsible %}

Deploy `src/Lab2/4-observability/docker-compose.yml` using the following command and issue some commands through the interface. Navigate to `localhost:9415` to access the Zipkin interface. In the "Dependencies" tab, take a wide range (e.g., [j-1, j+1]) and observe the diagram.

> **Question**: Which services are displayed? How do these services communicate?

Solution:
{% collapsible %}
The displayed services are:

- **command-api**
- **order-processing**
- **receipt-generator**
- **stock-manager**

Communications:

- **command-api** --> **order-processing**: pub/sub
- **order-processing** --> **receipt-generator**: service invocation
- **order-processing** --> **stock-manager**: service invocation
{% endcollapsible %}

> **Question**: What is/are the missing service(s)? Why?

**Hint**: There may be some waiting time before the diagram is displayed. You can use the image below if time is limited.

{% collapsible %}
![zipkin deps](/media/lab2/observability/zipkin-deps.png)
{% endcollapsible %}

Solution:
{% collapsible %}
The missing service is **command-frontend**. Indeed, it is the only service that does not have a sidecar, as it is located on the client side.

It is also noticeable that bindings are not represented. Bindings connect our application to external systems, which are not traced in a report on internal call chains.
{% endcollapsible %}

### Observing Metrics

Another aspect of observability is the "metrics" of containers. These metrics are general statistics that provide an operational view of the cluster.

These statistics include:

- CPU usage of sidecars
- RAM usage of sidecars
- Average latency between each service and its sidecar
- Uptime of sidecars
- Component states
- HTTP/gRPC call statistics

The complete list of metrics sent by each Dapr service is available [here](https://github.com/dapr/dapr/blob/master/docs/development/dapr-metrics.md).

All these metrics are emitted in [an open format](https://github.com/prometheus/docs/blob/main/content/docs/instrumenting/exposition_formats.md) and can be retrieved and analyzed by dedicated tools such as [Azure Monitor](https://azure.microsoft.com/en-us/services/monitor/) or [Prometheus](https://prometheus.io).

[Each release](https://github.com/dapr/dapr/releases) of Dapr also provides pre-designed Grafana dashboards for a graphical summary of the collected metrics.

![grafana dapr doc](/media/lab2/metrics/grafana-doc-example.png)

A guide is available [in the documentation](https://docs.dapr.io/operations/monitoring/metrics/grafana/) explaining the setup of these dashboards.

#### In Practice (Optional)

> **Important**: This section will focus on obtaining some metrics on the red thread example. However, it should be noted that the Grafana dashboards provided by the Dapr team are designed for Kubernetes, and some metrics may not be available.

To obtain metrics for our application, we need to add two services to our deployment: Prometheus and Grafana.

Prometheus is a time-series analysis tool (~= variables evolving over time). It can retrieve and aggregate information from *n* HTTP sources.

Grafana is a visualization tool often used in conjunction with Prometheus. It can create dashboards from multiple data sources, providing a detailed visualization of the data to the user.

> **In Practice**: Deploy the application specified by `src/Lab2/5-metrics/docker-compose.yml`. Now, navigate to Grafana at **localhost:9417**, then choose the "dapr-sidecar-dashboard" dashboard in _Dashboards -> Browse_. What do you observe?

**Hint**: You may need to reduce the observation window to 5 minutes to see changes in the graphs and issue some Charentaises commands.

Solution:
{% collapsible %}
![sample grafana result](/media/lab2/metrics/sample-grafana-result.png)

It can be observed that the present metrics include latencies and component metrics.

CPU/RAM usage is not specified for this example. Indeed, these data are extrapolated from [Kubernetes metrics](https://github.com/kubernetes/kubernetes/blob/master/test/instrumentation/testdata/stable-metrics-list.yaml), which are not present in a Docker Compose context.
{% endcollapsible %}
##### Out of Curiosity: Some Details on Prometheus and Docker Compose

The deployment of this section is not explicitly explained but executed. The reason for this is that making Prometheus work with Dapr on Docker Compose requires using a specific configuration that is not suitable for a production scenario.

Indeed, Dapr metrics are emitted by each sidecar on their respective port 9090 (by default). In the context of usage on Kubernetes or without an orchestrator, each sidecar would emit on the same endpoint on port 9090. In that case, it would only be necessary to specify this endpoint in Prometheus.

However, the handling of [docker networks](https://docs.docker.com/network/) in Docker Compose does not allow each sidecar to emit on the same endpoint. To achieve the desired behavior, it is necessary to explicitly list each service in the Prometheus configuration.

```yaml
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

Additionally, each (service, sidecar) pair shares the same network interface, so it is counterintuitively necessary to expose port 9090 of the service to reach this same port on the sidecar.

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

Once Prometheus is configured, Grafana does not pose any particular issues.

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

### Observing Logs

The last aspect of observability we will address is log observation. The ability to store and analyze logs is an integral part of the life of a distributed application – perhaps even more so than the previous aspects – and it is common for each developer to already have a more or less managed solution they are familiar with.

So, there is no discussion here about how the logs of the services themselves are treated; this part will focus only on **the logs of the sidecars**.

The support used for this example will be an ELK stack (Elasticsearch, Logstash, Kibana). However, this is not the only solution supported by Dapr.

> **In Practice**: Deploy the application specified by `src/Lab2/6-logs/docker-compose.yml`. Now navigate to Kibana at **localhost:5601**.

**Note**: The deployment may fail with a _Dial [1]::24224_ error; simply rerun the command in that case.

The Kibana instance is already configured to receive logs from sidecars (see the [**Details**](#details) section below). Now, you need to configure this instance to analyze them.

To do this:

- Once on the interface, choose the option _Explore on my own_ when prompted to add integrations.
- Click on the menu in the top left (**☰**) and then on _Stack Management_ in the _Management_ section.
- Once on the new page, in the left panel, click on _Data View_ in the _Kibana_ section.
- Click on _Create Data View_.
- You will be asked to enter an index. There should be only one available in the form **fluent-\<\>**. Enter "fluent\*" in the left field.
- Click on _Create Data View_.

By creating this view, you will have a list of variables included in the logs.

> **Question**: Comparing the displayed variables with [the Dapr log format](https://docs.dapr.io/operations/monitoring/logging/logs/#log-schema), what differences do you notice? Why?

**Solution**:

{% collapsible %}
![Dapr log items list](/media/lab2/logs/log-items-full.png)

Compared to the Dapr format, there are many additional variables. These variables come from the used [ECS format](https://www.elastic.co/guide/en/observability/8.3/logs-app-fields.html).
{% endcollapsible %}

Now, to view the logs, click again on **☰** and then click on _Discover_ in the _Analytics_ section.

**Note**: Duplicates with ".keyword" suffixes are noticed in the attributes. This is an Elasticsearch specificity: when encountering a string, Elasticsearch will index it both as _TEXT_, a field in which it is possible to search for a substring, and as _KEYWORD_, not indexed. However, it is possible to specify the behavior for each of the fields.

> **In Practice**: Isolate the logs of the **order-processing** container. Comment on the attributes.

{% collapsible %}

To isolate the logs of the **order-processing** container, simply search for "app-id: order-processing" in the search bar.

A log line has this format:

```jsonc
// Note: .keyword attributes are ignored
{
  "_index": "fluentd-20220704",
  "_id": "sepuyYEB3S4ZspuilsDO",
  "_version": 1,
  "_score": null,
  "fields": {
    // Dapr payload. Note that each of these attributes is
    // separately found in the "fields" object. This is an effect of FluentD configuration
    // that parses the JSON of this attribute and integrates it into the parent object
    "log": [
      "{\"app_id\":\"order-processing\",\"instance\":\"d900866e4786\",\"level\":\"info\",\"msg\":\"application configuration loaded\",\"scope\":\"dapr.runtime\",\"time\":\"2022-07-04T13:37:51.345159194Z\",\"type\":\"log\",\"ver\":\"edge\"}"
    ],
    // Payload -> Original file descriptor
    "source": ["stdout"],
    // Payload -> Actual log message
    "msg": ["application configuration loaded"],
    // Payload -> Log type
    "type": ["log"],
    "scope": ["dapr.runtime"],
    // Payload -> APP-id of the service emitting the log
    "app_id": ["order-processing"],
    // Payload -> Dapr version
    "ver": ["edge"],
    // Payload -> Log level
    "level": ["info"],
    // Meta -> Log reception time in timestamp format for log sorting
    "@timestamp": ["2022-07-04T13:37:51.345Z"],
    // Meta -> Container name as seen by docker compose
    "container_name": ["/6-logs_order-processing-dapr_1"],
    // Meta -> Docker container UUID
    "container_id": [
      "1f8457fe7a405ecf8443304558aedbeaa4bd9eff0a45ecd2b7aa24caf4879e73"
    ]
  },
  // Reception date in timestamp format to sort logs
  "sort": [1656941871345]
}
```

{% endcollapsible %}

#### Details

The way logs are sent to Kibana involves using [FluentD](https://docs.fluentd.org/) as the [logging driver](https://docs.docker.com/config/containers/logging/configure/).

Dapr is then configured to display logs in JSON format on stdout with the `-log-as-json` command option.

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

This method, specific to the Docker Compose orchestrator, is not detailed in the core activity.

On Kubernetes, however, the deployment would be similar. It would be sufficient to deploy FluentD on the cluster and add the `log-as-json` annotation to the deployment of services to achieve the same result. A complete tutorial is available [on the Dapr website](https://docs.dapr.io/operations/monitoring/logging/fluentd/).
