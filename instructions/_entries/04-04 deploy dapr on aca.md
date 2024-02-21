---
sectionid: lab3-deploy-dapr-on-aca
sectionclass: h2
title: Deploy a Dapr app on ACA
parent-id: lab-3
---

### Solution for the First Exercise

To create two container apps using the `az containerapp create` command for the provided services, you can use the following commands:

```bash
# Nodeapp
az containerapp create \
  --name nodeapp \
  --image dockerutils/nodeapp \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --ingress 'internal' \
  --min-replicas 1 \
  --target-port 3000 \
  --enable-dapr \
  --dapr-app-id 'nodeapp'

# Pythonapp
az containerapp create \
  --name pythonapp \
  --image dockerutils/pythonapp \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --min-replicas 1 \
  --enable-dapr \
  --dapr-app-id 'pythonapp'
```

This script will create two container apps named `nodeapp` and `pythonapp` using the provided Docker images. The `--enable-dapr` flag is used to enable Dapr for these container apps, and the `--dapr-app-id` flag sets the Dapr application ID.

### Solution for the Second Exercise

Looking at the logs of the `nodeapp` container, you'll notice an error related to the Dapr state store:

![Trace](/media/lab3/trace-q1.png)

This error is due to the fact that the Dapr `statestore` component is not declared. To resolve this, you need to create a Dapr component for the state store. However, the syntax for Dapr components is not fully supported on the Container Apps version used in this workshop.

Instead, you can use Azure Storage Account as the state store. Create a new Storage Account on Azure and retrieve the access key:

```bash
# Replace the placeholders with your values
az storage account create --name <STORAGE_ACCOUNT_NAME> --resource-group <RESOURCE_GROUP> --location <LOCATION> --sku Standard_LRS --kind StorageV2

# Get the access key for the created Storage Account
az storage account keys list --resource-group <RESOURCE_GROUP> --account-name <STORAGE_ACCOUNT_NAME> --query '[0].value' --out tsv
```

Create a YAML file for the state storage component:

```yaml
componentType: state.azure.blobstorage
version: v1
metadata:
  - name: accountName
    value: "<STORAGE_ACCOUNT_NAME>"
  - name: accountKey
    secretRef: account-key
  - name: containerName
    value: state
secrets:
  - name: account-key
    value: "<STORAGE_ACCOUNT_KEY>"
scopes:
  - nodeapp
```

Replace `<STORAGE_ACCOUNT_NAME>` and `<STORAGE_ACCOUNT_KEY>` with the appropriate values. Then, update the `nodeapp` container app to use the new state storage component:

```bash
az containerapp create \
  --name nodeapp \
  --image dockerutils/nodeapp \
  --resource-group $RESOURCE_GROUP \
  --environment $CONTAINERAPPS_ENVIRONMENT \
  --ingress 'internal' \
  --target-port 3000 \
  --enable-dapr \
  --dapr-app-id 'nodeapp'
```

Now, check the logs again for the `nodeapp` container to verify that it's working correctly.

### Final Note

This concludes the workshop. If you have any further questions or need clarification on any topic, feel free to ask!
