---
sectionid: lab3-intro
sectionclass: h2
title: Context
parent-id: lab-3
---

Lors des deux premiers *Labs*, nous nous sommes intéressés à Dapr en tant que logiciel indépendant de sa plateforme d'exécution.

Dapr peut en effet être exécuté en tant que processus local ou en tant que *sidecar* dans des orchestrateurs comme *docker-compose*, *docker-swarm* et évidemment *Kubernetes*. Ce que va permettre cette flexibilité, c'est d'ouvrir la voie à des services PaaS pour inclure Dapr en tant que fonctionnalité. Avec cette couche d'abstraction supplémentaire, les dernières problématiques "OPS" - par exemple s'assurer que le procesus de Dapr est présent pour chaque service - sont éliminées et on peut enfin considérer Dapr comme une solution totalement orientée DEV.

