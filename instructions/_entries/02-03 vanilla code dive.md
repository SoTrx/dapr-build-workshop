---
sectionid: lab1-codedive
sectionclass: h2
title: Plonger dans le code
parent-id: lab-1
---

```markdown
Looking at the code of the Node and Python services, answer the following questions:

> **Question**: How do the three services communicate with each other?

Solution:
{% collapsible %}
The three services communicate **directly**:

- The Python service calls the Node service via an **HTTP request** to its server's URL.
- The Node service calls Redis using the **[ioredis](https://www.npmjs.com/package/ioredis)** library, which encapsulates the **RESP** protocol, Redis's specific protocol.
  {% endcollapsible %}

Imagine that this application has been deployed in production for some time. However, after a few months, a new requirement emerges: the state storage support needs to be migrated from Redis to MongoDB.

> **Question**: What would be the impact of this change on the NodeJS and Python services? Propose a protocol to perform this migration.

Solution:
{% collapsible %}
The call from the Python service to the Node service would not be affected.

However, the Nodejs application code would need to be rewritten.
Indeed, to communicate with Redis, the service uses the **[ioredis](https://www.npmjs.com/package/ioredis)** library.
This library would no longer be suitable in the code if the implementation changes from Redis to MongoDB.

This is one of the consequences of a **strong application coupling**.
{% endcollapsible %}
```