---
sectionid: lab2-secrets
sectionclass: h2
title: Secret management
parent-id: lab-2
---
Translate to English while maintaining the format:

In all the scenarios we have seen so far, for simplicity, we have used services without any form of authentication. In a production scenario, however, it will be essential to engage in proper secret management.

### Generalities

> **General question**: What is a digital safe (or secret manager)? What is the advantage of storing secrets in this safe instead of directly providing them in the services' environment?

Solution:

{% collapsible %}

A secret manager is a service that centralizes the creation/retrieval/deletion of secrets for a distributed application.

Although adding an additional indirection, this solution has essential advantages such as:

- Preventing the duplication of a secret used multiple times
- Allowing a form of authentication/authorization, controlling which services access which secrets
- Keeping a record of accesses, facilitating security audits

{% endcollapsible %}

### Dapr

Using the [documentation](https://docs.dapr.io/developing-applications/building-blocks/secrets/secrets-overview/), answer the following questions:

> **Question**: What are the different safes supported by Dapr? Which ones are usable in production?

Solution:

{% collapsible %}

According to [the documentation](https://docs.dapr.io/reference/components-reference/supported-secret-stores/) (February 2023), the different supported safes are:

| Name                                   | Component | Location                 | Production?                                                          |
| -------------------------------------- | --------- | ------------------------ | --------------------------------------------------------------------- |
| Hashicorp Vault                        | Stable    | External/In-cluster       | Yes                                                                   |
| Secrets Kubernetes                     | Stable    | In-cluster               | Avoid. Secrets in base64, limited to a namespace                      |
| Environment Variables                  | Stable    | In the node              | No                                                                    |
| File                                   | Stable    | In the container         | No                                                                    |
| Alibaba Parameter Store                | Alpha     | External                 | Yes. Pay attention to the component's maturity                         |
| AWS Secrets Manager / Parameter Store  | Alpha     | External                 | Yes. Pay attention to the component's maturity                         |
| GCP Parameter Store                    | Alpha     | External                 | Yes. Pay attention to the component's maturity                         |
| Azure Key Vault                        | Stable    | External                 | Yes                                                                   |

In general, it will always be preferable to store secrets outside the service execution platform to avoid "putting all your eggs in one basket." In case the secret manager is hosted on the same platform as the services, attention should be paid to the reliability of this extremely critical part (cluster mode, persistence, backups, etc.).

{% endcollapsible %}

> **Question**: What are the two ways to access the secret manager through Dapr?

Solution:

{% collapsible %}
The documentation page presents two ways to access the secret manager through Dapr:

- Use the Dapr API REST / SDKs from the **services** themselves
- Use references to secrets from declared **components**

#### From Services

From the service code, you just need to use the corresponding API:

```curl
GET/POST http://localhost:3500/v1.0/secrets/<COMPONENT_NAME>/<SECRET_NAME>
```

where:

- **\<COMPONENT_NAME\>**: Name of the secret management component
- **\<SECRET_NAME\>**: Key/Namespace (depending on the underlying storage) of the stored secret

#### From Components

The other way is to modify existing Dapr components to include references to the secret manager.

```diff
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: statestore
spec:
  type: state.redis
  version: v1
  metadata:
  - name: redisHost
    value: localhost:6379
  - name: redisPassword
    # Instead of using a value 
-    value: ""
    # A reference is passed
+    secretKeyRef:
+    	name: <NAMESPACE>
+     key:  <SECRET_KEY>
+auth:
+  secretStore: <SECRET_STORE_COMPONENT>

```

{% endcollapsible %}

> **Question**: Propose a Dapr configuration for the **service-1** allowing it to only access the **secret-1** in the secret management component **vault**.
Solution:

{% collapsible %}

To achieve this, you would simply apply this configuration (see monitoring section) to the **service-1**.

```yml
apiVersion: dapr.io/v1alpha1
kind: Configuration
metadata:
  name: appconfig
spec:
  secrets:
    scopes:
      - storeName: vault
        defaultAccess: deny
        allowedSecrets: ["secret-1"]
```

However, it's important to note that other services would have access to all secrets, which could pose a problem.

Another possible configuration in this case would be to apply a configuration to all services that, by default, denies access to all secrets. Similar to a firewall operation, each service would then need to be explicitly allowed to access each secret.

{% endcollapsible %}

### In Application

The red thread application continues to evolve. This time, the Redis instance used for communication between `command-api` and `order-processing` has been modified to require a password: **suchStrongP4ssword!!**

A new service has also been added to the application: a [HashiCorp Vault](https://www.vaultproject.io/). This vault has been initialized (in [development mode](https://www.vaultproject.io/docs/concepts/dev-server)) as follows:

| Namespace | Value                           |
| --------- | ------------------------------- |
| redis     | REDIS_PASS=suchStrongP4ssword!! |

This vault is accessible in the application at `http://vault:8200`. The access token (_root token_) is **roottoken**.

> **In practice**: Using the information below, declare the Dapr component (in the `src/Lab2/7-secrets/components` folder) corresponding to the Vault.

**Hint**: Remember to disable TLS verification.

Solution:

{% collapsible %}

Since the used vault is a HashiCorp Vault, the corresponding [Dapr component](https://docs.dapr.io/reference/components-reference/supported-secret-stores/hashicorp-vault/) should be used.

This component has many parameters, but since the deployment of the vault in this case is straightforward, only three parameters are truly required.

The component to create is:

```yml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  # The Dapr component named "vault"...
  name: vault
spec:
  # is an instance of HashiCorp Vault...
  type: secretstores.hashicorp.vault
  version: v1
  metadata:
    # available at the address http://vault:8200...
    - name: vaultAddr
      value: "http://vault:8200"
    # not using TLS...
    - name: skipVerify
      value: "false"
    # with the password "roottoken"
    - name: vaultToken
      value: "roottoken"
```

{% endcollapsible %}

With the component created, it is possible to retrieve secrets in the two ways we have discussed earlier.

> **In practice**: Modify the `pubsub.yml` component to include retrieving the password from the secret management component. Check the functionality by running the application located at `src/Lab2/7-secrets/docker-compose.yml`

**Hint**: In case of failure to retrieve the password from the Redis instance, some services might simply crash.

Solution:

{% collapsible %}

```yml
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
    # The password is a secret...
    - name: redisPassword
      secretKeyRef:
        # located in the "redis" namespace
        name: redis
        # and is the value corresponding to the REDIS_PASS key
        key: REDIS_PASS
# In the secret management component called "vault"
auth:
  secretStore: vault
```

{% endcollapsible %}

### Conclusion

Secret management is such an essential aspect of the life of a distributed application that it must be included from its conception.

In this example, we use a secret manager directly integrated into the application, via a Vault. While this configuration is sufficient for an exercise, it is not recommended in this form for production.

In a fully private production configuration, it would be necessary to provide a persistence layer for the secret manager and better access security.

In a configuration using public cloud services, it would be more advisable to turn to a managed service, such as [Azure Keyvault](https://azure.microsoft.com/en-us/services/key-vault/).
