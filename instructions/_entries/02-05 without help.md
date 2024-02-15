---
sectionid: lab1-withouthelp
sectionclass: h2
title: Le mode processus (Facultatif)
parent-id: lab-1
---

```markdown
In the previous exercises, we mostly worked with containers, and even a `docker-compose.yml` file was provided.

However, it is possible to use Dapr in process mode, which allows for a simpler development/debugging process.

> **Practical Exercise**: Install [Dapr locally on your PC](https://docs.dapr.io/getting-started/install-dapr-cli/) and initialize your local environment.

**Hint**: [Documentation](https://docs.dapr.io/getting-started/). <u>Do not initialize Dapr for a Kubernetes environment!</u>

> **Question**: What containers are deployed by the initialization of Dapr? What is the role of each?

Solution:
{% collapsible %}
The deployed containers are:

- Redis: By initializing Dapr, an instance of Redis and the associated component are created (in the `~/.dapr/components` directory). Redis can both handle state persistence and message distribution, making it often chosen as the default component **in development environments**.
- Zipkin: An [Open-Source software](https://github.com/openzipkin/zipkin) for tracing calls between services (See _Lab2_)
- Placement: A Dapr service allowing the use of the [distributed programming model "Actors"](https://docs.dapr.io/developing-applications/building-blocks/actors/actors-overview/). A detailed explanation of the principles of this model can be found on the [GitHub of the **Orleans** project](https://github.com/dotnet/orleans).
  {% endcollapsible %}

### Deploying an Application in Process Mode

Now, we will deploy the application located in the `src/Lab2/1-without-help` folder in process mode.

Since the deployment type we are aiming for is not containerized, we first need to install the software stacks (_stacks_) for each service:

- Nodejs Application:
  - Install `nodejs` (>= 8.0)
  - Install application dependencies with `npm install` (in the `src/Lab2/1-without-help/node` directory)
  - Transpile TypeScript into JavaScript with the command `npm run build` (in the `src/Lab2/1-without-help/node` directory)
  - The command to run the application is `node dist/app.js`.
- Python Service:
  - Install `python` (>= 3.0)
  - (The application has no dependencies)
  - The command to run the application is `python3 app.py`

> **Practical Exercise**: Run it in process mode using Dapr's `dapr run` command.

**Hint**: You can refer to this [quick start documentation](https://docs.dapr.io/getting-started/quickstarts/pubsub-quickstart/).

Solution:
{% collapsible %}
After completing the above steps, simply run the following commands:

```shell
# cwd: src/Lab2/1-without-help/node
dapr run --app-id nodeapp --components-path ../components --app-port 3000 -- node dist/app.js
# cwd: src/Lab2/1-without-help/python (in another shell)
# Note: On Linux, the python executable is sometimes still called python3
dapr run --app-id pythonapp --components-path ../components -- python app.py
```

{% endcollapsible %}

### Summary

Dapr allows both assisting containerized applications (sidecar pattern) and process mode applications.

While the sidecar part is indicated in cases of container orchestration (Kubernetes, docker-compose, docker swarm, etc.), the process part allows us to run applications locally without having to think about containerizing them.

These two operating modes ensure compatibility with cloud services such as managed Kubernetes (AKS) or simply hosting applications in process mode (App Service).
```