---
sectionid: lab3-create-aca
sectionclass: h2
title: Create a new ACA environment
parent-id: lab-3
---

### Solution for Creating an Environment and Deploying a Service

Before creating a service, let's create a new environment for Azure Container Apps. We'll use the Azure CLI with the following commands:

```bash
# Replace <Resource_Group_Name> and <Environment_Name> with your desired names
az group create -n <Resource_Group_Name> --location westeurope
az containerapp env create -n <Environment_Name> -g <Resource_Group_Name> --location westeurope
```

These commands create a new resource group and a new environment within that resource group. Ensure that you replace `<Resource_Group_Name>` and `<Environment_Name>` with your preferred names.

Once the environment is created, we can proceed to deploy a service. For this example, we'll deploy a simple hello-world service using the Azure CLI.

```bash
# Replace <Service_Name> and <Image_Name> with your desired names
az containerapp create -n <Service_Name> --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest -e <Environment_Name> --rg <Resource_Group_Name> --ingress external --target-port 80 --os-type Linux
```

Replace `<Service_Name>` with the desired name for your service, and `<Resource_Group_Name>` and `<Environment_Name>` with the names you used while creating the resource group and environment.

This command deploys a hello-world service (`mcr.microsoft.com/azuredocs/containerapps-helloworld:latest`) with the specified settings. It uses an external ingress, exposes the service on port 80, and specifies Linux as the operating system.

After the deployment is successful, you will receive information about the deployed service, including its ID, name, location, and configuration details.

Remember to replace placeholder values with your actual names.
