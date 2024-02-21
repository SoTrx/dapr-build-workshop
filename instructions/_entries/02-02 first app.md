---
sectionid: first-app
sectionclass: h2
title: A first application
parent-id: lab-1
---

> **Note**: The files used in this initial application are located in the `src/Lab1/1-decoupling/direct` folder.

### A First Application

The focus of this first activity is the application represented below.

![First App](/media/lab1/first-app-vanilla.png)

This application consists of three parts:

- An instance of Redis to allow state storage.
- A Node service that stores state in Redis.
- A Python service that generates and sends state to the Node service every second.

### Starting the Application Locally

The application can be run locally using docker-compose.

```shell
    # The "--no-attach" option hides the Redis logs
    # If there is an issue, it can be removed to get more details
    docker-compose up --no-attach redis
```

> **Practical Exercise**: Start the application. Verify that you get a trace like the one below:

![Results](/media/lab1/first-app-vanilla-result.png)