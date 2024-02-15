---
sectionid: lab3-hello-world-aca
sectionclass: h2
title: A first app on ACA
parent-id: lab-3
---
### Solution for the Third Exercise

To deploy a simple hello-world application on Azure Container Apps, you can use the following command:

```bash
az containerapp create \
  --name my-container-app \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --ingress 'external' \
  --target-port 80 \
  --query configuration.ingress.fqdn
```

This script creates a container app named `my-container-app` using the hello-world image from the Azure Container Apps registry. It exposes the container on the internet (`external` ingress) on port 80 and returns the Fully Qualified Domain Name (FQDN) of the container.

If you didn't use the `--query` option, the output will include various information about the created container, including its ID, location, name, properties, and more.

You can access the hello-world application by navigating to the provided FQDN in your browser. The application should display a simple "Hello, World!" message.

In the Azure portal, you can explore the details of the deployed container app. This includes configuration settings, the current revision, auto-scaling information, and more.

This exercise demonstrates the basic steps to deploy a container app on Azure Container Apps. Feel free to explore the Azure portal to discover additional metrics, logs, and capabilities provided by Azure Container Apps for managing and monitoring containerized applications.
