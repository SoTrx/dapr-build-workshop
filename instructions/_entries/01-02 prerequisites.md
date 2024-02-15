---
sectionid: prereq
sectionclass: h2
title: Prerequisites
parent-id: intro
---

### Prerequisites

This workshop requires the following:

- An Azure subscription
- Azure CLI (**>= 2.56**) **and its extension** for Containers Apps
- [Docker](https://www.docker.com/) with its [compose](https://docs.docker.com/compose/install/) extension (**>=1.27.0**). The extension is included in Docker-Desktop, so you only need to install it on Linux.
- [Workshop source code](https://aka.ms/daprartifacts)

### Install Azure CLI and the extension for Azure Container Apps

#### If CLI is not installed: Install the CLI

Follow [this link](https://docs.microsoft.com/fr-fr/cli/azure/install-azure-cli) and choose the tab that corresponds to your operating system.

#### If CLI is installed: Upgrade its version

```bash
az version
# If az-core version <= 2.56
az upgrade
```

#### Install Azure Container Apps extension

Once the CLI is installed, install the extension for Azure Container Apps CLI

```bash
az extension add -n containerapp
```

#### Log in to your subscription

Finally, connect to your subscription using the following command

```bash
az login
```